---
layout: post
title: Supervisor
description: 
category: storm
---

supervisor相当于单机版的executor调度器，他会听从nimbus的调度。

supervisor的元数据如下：

	(defn supervisor-data [conf shared-context ^ISupervisor isupervisor]
	  {:conf conf
	   :shared-context shared-context;;上下文相关信息，现在为Nil
	   :isupervisor isupervisor;;isupervisor的实现
	   :active (atom true);;supervisor的运行状态
	   :uptime (uptime-computer);;迄今为止的运行时间
	   :worker-thread-pids-atom (atom {});;Supervisor启动的所有worker的进程号集合
	   :storm-cluster-state (cluster/mk-storm-cluster-state conf);;storm-cluster-state在分布式下是可以和ZK交互，获得集群信息
	   :local-state (supervisor-state conf);;LocalState
	   :supervisor-id (.getSupervisorId isupervisor);;id
	   :assignment-id (.getAssignmentId isupervisor);;id
	   :my-hostname (if (contains? conf STORM-LOCAL-HOSTNAME)
                  (conf STORM-LOCAL-HOSTNAME)
                  (local-hostname));;主机名的信息，可以配置
	   :curr-assignment (atom nil) ;;记录了本机使用的端口好，心跳时会用到
	   :timer (mk-timer :kill-fn (fn [t]
                               (log-error t "Error when processing event")
                               (halt-process! 20 "Error when processing an event")
                               ));;timer
	   })



现在来看看他是怎么运行的。

	(defn -main []
	  (-launch (standalone-supervisor)))

	(defn standalone-supervisor []
	  (let [conf-atom (atom nil);;conf的引用
        id-atom (atom nil)];;id的引用
	    (reify ISupervisor;;实现了ISupervisor接口
	      (prepare [this conf local-dir];;实现prepare方法
	        (reset! conf-atom conf);;重置conf属性
	        (let [state (LocalState. local-dir);;创建一个LocalState
	              curr-id (if-let [id (.get state LS-ID)]
	                        id
	                        (generate-supervisor-id))];;如果不存在就生成一个supervisorId
	          (.put state LS-ID curr-id);;id保存到state
	          (reset! id-atom curr-id));;id保存到id-atom
	        )
	      (confirmAssigned [this port]
	        true)
	      (getMetadata [this]
	        (doall (map int (get @conf-atom SUPERVISOR-SLOTS-PORTS))))
	      (getSupervisorId [this]
	        @id-atom)
	      (getAssignmentId [this]
	        @id-atom)
	      (killedWorker [this port]
	        )
	      (assigned [this ports]
        ))))

standalone-supervisor对象生成完毕后，来看看launch方法：
(defn supervisor-data [conf shared-context ^ISupervisor isupervisor]
  {:conf conf
   :shared-context shared-context
   :isupervisor isupervisor
   :active (atom true)
   :uptime (uptime-computer)
   :worker-thread-pids-atom (atom {})
   :storm-cluster-state (cluster/mk-storm-cluster-state conf)
   :local-state (supervisor-state conf)
   :supervisor-id (.getSupervisorId isupervisor)
   :assignment-id (.getAssignmentId isupervisor)
   :my-hostname (if (contains? conf STORM-LOCAL-HOSTNAME)
                  (conf STORM-LOCAL-HOSTNAME)
                  (local-hostname))
   :curr-assignment (atom nil) ;; used for reporting used ports when heartbeating
   :timer (mk-timer :kill-fn (fn [t]
                               (log-error t "Error when processing event")
                               (halt-process! 20 "Error when processing an event")
                               ))
   })
	(defn -launch [supervisor]
	  (let [conf (read-storm-config)];;从文件中读取conf信息
	    (validate-distributed-mode! conf);;验证conf必须为分布式设置
	    (mk-supervisor conf nil supervisor)));;创建supervisor


mk-supervisor方法如下：

	(defserverfn mk-supervisor [conf shared-context ^ISupervisor isupervisor]
	  (log-message "Starting Supervisor with conf " conf)
	  (.prepare isupervisor conf (supervisor-isupervisor-dir conf));;调用isupervisor的prepare方法，储存conf
	  (FileUtils/cleanDirectory (File. (supervisor-tmp-dir conf)));;清空supervisor的tmp目录
	  (let [supervisor (supervisor-data conf shared-context isupervisor);;构建supervisor-data
        [event-manager processes-event-manager :as managers] [(event/event-manager false) (event/event-manager false)];;创建了两个event-manager，用来处理事件的，event-manager会把事件暂存在queue中，然后单线程串行的处理。                      
        sync-processes (partial sync-processes supervisor);;用来管理worker进程的线程
        synchronize-supervisor (mk-synchronize-supervisor supervisor sync-processes event-manager processes-event-manager);;用来同步nimbus任务的线程
        heartbeat-fn (fn [] (.supervisor-heartbeat!
                               (:storm-cluster-state supervisor)
                               (:supervisor-id supervisor)
                               (SupervisorInfo. (current-time-secs)
                                                (:my-hostname supervisor)
                                                (:assignment-id supervisor)
                                                (keys @(:curr-assignment supervisor))
                                                ;; used ports
                                                (.getMetadata isupervisor)
                                                (conf SUPERVISOR-SCHEDULER-META)
                                                ((:uptime supervisor)))))];;心跳线程
    (heartbeat-fn)
    ;; should synchronize supervisor so it doesn't launch anything after being down (optimization)
    (schedule-recurring (:timer supervisor)
                        0
                        (conf SUPERVISOR-HEARTBEAT-FREQUENCY-SECS)
                        heartbeat-fn);;定时发心跳
    (when (conf SUPERVISOR-ENABLE)
      ;; This isn't strictly necessary, but it doesn't hurt and ensures that the machine stays up
      ;; to date even if callbacks don't all work exactly right
      (schedule-recurring (:timer supervisor) 0 10 (fn [] (.add event-manager synchronize-supervisor)));;定期执行synchronize-supervisor
      (schedule-recurring (:timer supervisor)
                          0
                          (conf SUPERVISOR-MONITOR-FREQUENCY-SECS)
                          (fn [] (.add processes-event-manager sync-processes))));;定期执行sync-processes
    (log-message "Starting supervisor with id " (:supervisor-id supervisor) " at host " (:my-hostname supervisor))
    (reify
     Shutdownable
     (shutdown [this]
               (log-message "Shutting down supervisor " (:supervisor-id supervisor))
               (reset! (:active supervisor) false)
               (cancel-timer (:timer supervisor))
               (.shutdown event-manager)
               (.shutdown processes-event-manager)
               (.disconnect (:storm-cluster-state supervisor)))
     SupervisorDaemon
     (get-conf [this]
       conf)
     (get-id [this]
       (:supervisor-id supervisor))
     (shutdown-all-workers [this]
       (let [ids (my-worker-ids conf)]
         (doseq [id ids]
           (shutdown-worker supervisor id)
           )))
     DaemonCommon
     (waiting? [this]
       (or (not @(:active supervisor))
           (and
            (timer-waiting? (:timer supervisor))
            (every? (memfn waiting?) managers)))
           ))))

用来管理worker进程的函数sync-processes：

	(defn sync-processes [supervisor]
	  (let [conf (:conf supervisor);;获得conf
        ^LocalState local-state (:local-state supervisor);;获得local-state
        assigned-executors (defaulted (.get local-state LS-LOCAL-ASSIGNMENTS) {});;从local-state中获取LS-LOCAL-ASSIGNMENTS属性，是个<port, Assignment>集合，如果没有，就返回空
        now (current-time-secs);;当前时间
        allocated (read-allocated-workers supervisor assigned-executors now);;从localState中获取已经分配的worker信息包括状态和心跳信息
        keepers (filter-val
                 (fn [[state _]] (= state :valid))
                 allocated);;对allocated进行过滤，只保留state=:valid的worker
        keep-ports (set (for [[id [_ hb]] keepers] (:port hb)));;获得保留的端口信息
        reassign-executors (select-keys-pred (complement keep-ports) assigned-executors);;从已经分配的executor中根据keep-ports找出需要重新分配的executor
        new-worker-ids (into
                        {}
                        (for [port (keys reassign-executors)]
                          [port (uuid)]))
        ];;返回一个port对workerid的映射
    ;; 1. to kill are those in allocated that are dead or disallowed
    ;; 2. kill the ones that should be dead
    ;;     - read pids, kill -9 and individually remove file
    ;;     - rmr heartbeat dir, rmdir pid dir, rmdir id dir (catch exception and log)
    ;; 3. of the rest, figure out what assignments aren't yet satisfied
    ;; 4. generate new worker ids, write new "approved workers" to LS
    ;; 5. create local dir for worker id
    ;; 5. launch new workers (give worker-id, port, and supervisor-id)
    ;; 6. wait for workers launch
  
    (log-debug "Syncing processes")
    (log-debug "Assigned executors: " assigned-executors)
    (log-debug "Allocated: " allocated)
    (doseq [[id [state heartbeat]] allocated];;遍历分配的worker
      (when (not= :valid state);;如果不是:valid
        (log-message
         "Shutting down and clearing state for id " id
         ". Current supervisor time: " now
         ". State: " state
         ", Heartbeat: " (pr-str heartbeat))
        (shutdown-worker supervisor id);;关闭worker
        ))
    (doseq [id (vals new-worker-ids)];;需要重新分配的worker
      (local-mkdirs (worker-pids-root conf id)));;创建他们的目录
    (.put local-state LS-APPROVED-WORKERS 
          (merge
           (select-keys (.get local-state LS-APPROVED-WORKERS)
                        (keys keepers));;从LS-APPROVED-WORKERS过滤出state=:valid的worker
           (zipmap (vals new-worker-ids) (keys new-worker-ids));;需要重新分配的worker
           ));;合并之后保存在local-state LS-APPROVED-WORKERS 中
    (wait-for-workers-launch ;;等待worker启动起来
     conf
     (dofor [[port assignment] reassign-executors]
       (let [id (new-worker-ids port)]
         (log-message "Launching worker with assignment "
                      (pr-str assignment)
                      " for this supervisor "
                      (:supervisor-id supervisor)
                      " on port "
                      port
                      " with id "
                      id
                      )
         (launch-worker supervisor
                        (:storm-id assignment)
                        port
                        id);;启动需要重新分配的worker
         id)))
    ))

同步Nimbus任务的函数mk-synchronize-supervisor，会定时与nimbus做同步，及时获得新任务，关停旧任务，Supervisor每次会把同步后的LocalAssignment信息更新到LocalState中。
	
	(defn mk-synchronize-supervisor [supervisor sync-processes event-manager processes-event-manager]
	  (fn this []
    (let [conf (:conf supervisor)
          storm-cluster-state (:storm-cluster-state supervisor)
          ^ISupervisor isupervisor (:isupervisor supervisor)
          ^LocalState local-state (:local-state supervisor)
          sync-callback (fn [& ignored] (.add event-manager this));;这个回调会简单的把mk-synchronize-supervisor再次加入到event-manager进行异步调用
          assignments-snapshot (assignments-snapshot storm-cluster-state sync-callback);;绑定zk上ASSIGNMENTS-SUBTREE目录子节点变更时的回调，并返回全部topology部署情况的快照
          storm-code-map (read-storm-code-locations assignments-snapshot);;获得<storm-id, master-code-location>的对应关系
          downloaded-storm-ids (set (read-downloaded-storm-ids conf));;获取当前已下载的topology所对应的storm-id信息
          all-assignment (read-assignments
                           assignments-snapshot
                           (:assignment-id supervisor));;通过filter筛选出分配给当前supervisor的任务信息:<port, LocalAssignment>
          new-assignment (->> all-assignment
                              (filter-key #(.confirmAssigned isupervisor %)));;通过isupervisor的confirmAssigned方法过滤出确定分配的assignment，不过当前实现什么都没做，待扩展
          assigned-storm-ids (assigned-storm-ids-from-port-assignments new-assignment);;返回分配的storm-id
          existing-assignment (.get local-state LS-LOCAL-ASSIGNMENTS)];;从local-state中找出现存的assignment
      (log-debug "Synchronizing supervisor")
      (log-debug "Storm code map: " storm-code-map)
      (log-debug "Downloaded storm ids: " downloaded-storm-ids)
      (log-debug "All assignment: " all-assignment)
      (log-debug "New assignment: " new-assignment)
      
      ;; download code first
      ;; This might take awhile
      ;;   - should this be done separately from usual monitoring?
      ;; should we only download when topology is assigned to this supervisor?
      (doseq [[storm-id master-code-dir] storm-code-map]
        (when (and (not (downloaded-storm-ids storm-id))
                   (assigned-storm-ids storm-id));;如果存在这个topology但是当前没有分配过
          (log-message "Downloading code for storm id "
             storm-id
             " from "
             master-code-dir)
          (download-storm-code conf storm-id master-code-dir);;从nimbus下载
          (log-message "Finished downloading code for storm id "
             storm-id
             " from "
             master-code-dir)
          ))

      (log-debug "Writing new assignment "
                 (pr-str new-assignment))
      (doseq [p (set/difference (set (keys existing-assignment))
                                (set (keys new-assignment)))];;对比新分配的和已经存在的部署，筛选出不同的worker
        (.killedWorker isupervisor (int p)));;杀掉这些worker，不过killedWorker现在没有实现
      (.assigned isupervisor (keys new-assignment));;isupervisor重新分配，不过assigned现在没有实现
      (.put local-state
            LS-LOCAL-ASSIGNMENTS
            new-assignment);;设置 local-state的LS-LOCAL-ASSIGNMENTS为new-assignment
      (reset! (:curr-assignment supervisor) new-assignment);;重设new-assignment
      ;; remove any downloaded code that's no longer assigned or active
      ;; important that this happens after setting the local assignment so that
      ;; synchronize-supervisor doesn't try to launch workers for which the
      ;; resources don't exist
      (if on-windows? (shutdown-disallowed-workers supervisor));;如果是windows，则关闭所有disallowed worker
      (doseq [storm-id downloaded-storm-ids]
        (when-not (assigned-storm-ids storm-id)
          (log-message "Removing code for storm id "
                       storm-id)
          (try
            (rmr (supervisor-stormdist-root conf storm-id))
            (catch Exception e (log-message (.getMessage e))))
          ));;移除已经下载的但不在assigned-storm-ids的topology信息 
      (.add processes-event-manager sync-processes);;调用sync-processes同步wokrer进程
      )))




