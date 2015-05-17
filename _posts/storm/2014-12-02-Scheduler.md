---
layout: post
title: Scheduler
description: 
category: storm
---

storm中共分为三种Scheduler:

 1. EvenScheduler 会将系统资源均匀的分配给多个Topology
 2. DefaultScheduler 首先释放掉其他Topology不需要的资源，之后调用EvenScheduler的方法进行资源分配
 3. IsolationScheduler 可以单独对某些Topology制定使用多少台机器，IsolationScheduler会优先对这些机器进行资源分配，这些Topology的资首先源分配完毕后再调用DefaultScheduler进行资源分配

##如何创建Scheduler

还记得分布式启动nimbus时会创建standalone-nimbus，他实现了INimbus接口，不过在实现getForcedScheduler方法时把返回值设为Nil。

不过，在mk-scheduler时会真正创建Scheduler：

	(defn mk-scheduler [conf inimbus]
	  (let [forced-scheduler (.getForcedScheduler inimbus)
        scheduler (cond
                    forced-scheduler
                    (do (log-message "Using forced scheduler from INimbus " (class forced-scheduler))
                        forced-scheduler);;首先会尝试直接从INimbus中获取
    
                    (conf STORM-SCHEDULER);;如果INimbus中没有，则根据conf中的STORM-SCHEDULER配置向中创建
                    (do (log-message "Using custom scheduler: " (conf STORM-SCHEDULER))
                        (-> (conf STORM-SCHEDULER) new-instance))
    
                    :else
                    (do (log-message "Using default scheduler");;如果conf中也没有相关配置，则会使用默认的DefaultScheduler
                        (DefaultScheduler.)))]
    (.prepare scheduler conf);;而后执行scheduler的prepare函数进行准备工作
    scheduler
    ))


##IScheduler
IScheduler接口总共定义了两个方法：

 1. void prepare(Map conf); //准备工作
 2. void schedule(Topologies topologies, Cluster cluster); //为集群内的Topology分配资源

##EvenScheduler

EvenScheduler是最基础的Scheduler，所以先来看看他是如何实现的。

	
	(defn -prepare [this conf]
	  )

prepare方法是空实现，说明EvenScheduler不用做任何准备工作

	
	(defn -schedule [this ^Topologies topologies ^Cluster cluster]
	  (schedule-topologies-evenly topologies cluster))

schedule方法会调用函数schedule-topologies-evenly：

	(defn schedule-topologies-evenly [^Topologies topologies ^Cluster cluster]
	  (let [needs-scheduling-topologies (.needsSchedulingTopologies cluster topologies)];;从cluster中查找需要被调度的topology,分为两个方面：1.实际的worker数低于预想worker数。2.worker数满足要求，但是实际的executor数少于预想的executor数
    (doseq [^TopologyDetails topology needs-scheduling-topologies
            :let [topology-id (.getId topology)
                  new-assignment (schedule-topology topology cluster);;调度topology并返回新分配的资源
                  node+port->executors (reverse-map new-assignment)]];;获取worker与executor的对应关系
      (doseq [[node+port executors] node+port->executors
              :let [^WorkerSlot slot (WorkerSlot. (first node+port) (last node+port));;创建WorkerSlot
                    executors (for [[start-task end-task] executors]
                                (ExecutorDetails. start-task end-task))]];;创建ExecutorDetails
        (.assign cluster slot topology-id executors)))));;分配资源

调度topology的方法：

	(defn- schedule-topology [^TopologyDetails topology ^Cluster cluster]
	  (let [topology-id (.getId topology)
        available-slots (->> (.getAvailableSlots cluster);;获得所有可用的WorkerSlot
                             (map #(vector (.getNodeId %) (.getPort %))));;转成vector的格式
        all-executors (->> topology
                          .getExecutors
                          (map #(vector (.getStartTask %) (.getEndTask %)))
                          set);;获取executor并转为#{[startTask, endTask]}的格式
        alive-assigned (get-alive-assigned-node+port->executors cluster topology-id);;获取存活的node+port对executor的对应关系
        total-slots-to-use (min (.getNumWorkers topology)
                                (+ (count available-slots) (count alive-assigned)));;获取Topology所能使用的全部slot
        reassign-slots (take (- total-slots-to-use (count alive-assigned))
                             (sort-slots available-slots));;将要分配的slot数
        reassign-executors (sort (set/difference all-executors (set (apply concat (vals alive-assigned)))));;将要分配的executor
        reassignment (into {}
                           (map vector
                                reassign-executors
                                ;; for some reason it goes into infinite loop without limiting the repeat-seq
                                (repeat-seq (count reassign-executors) reassign-slots)))]
    (when-not (empty? reassignment)
      (log-message "Available slots: " (pr-str available-slots))
      )
    reassignment));;如果存在，则返回

Cluster类中的assign方法如下：

	public void assign(WorkerSlot 和slot, String topologyId, Collection<ExecutorDetails> executors) {
        if (this.isSlotOccupied(slot)) {//判断slot是否已经被占
            throw new RuntimeException("slot: [" + slot.getNodeId() + ", " + slot.getPort() + "] is already occupied.");
        }
        
        SchedulerAssignmentImpl assignment = (SchedulerAssignmentImpl)this.getAssignmentById(topologyId);//获取这个topology的部署信息
        if (assignment == null) {//如果没有则创建
            assignment = new SchedulerAssignmentImpl(topologyId, new HashMap<ExecutorDetails, WorkerSlot>());
            this.assignments.put(topologyId, assignment);
        } else {//如果存在则检查要分配的这些executor是否已经被当前topology部署过了
            for (ExecutorDetails executor : executors) {
                 if (assignment.isExecutorAssigned(executor)) {
                     throw new RuntimeException("the executor is already assigned, you should unassign it before assign it to another slot.");
                 }
            }
        }

        assignment.assign(slot, executors);//部署
    }

	class SchedulerAssignmentImpl...
	
	public void assign(WorkerSlot slot, Collection<ExecutorDetails> executors) {
        for (ExecutorDetails executor : executors) {//部署过程就是把executor和slot放入当前保存的状态中
            this.executorToSlot.put(executor, slot);
        }
    }

	

##DefaultScheduler

DefaultScheduler作为默认的调度器他与EvenScheduler最大的区别就是会首先计算bad-slots，default-schedule函数与EvenScheduler中的schedule-topology几乎一样：

	(defn default-schedule [^Topologies topologies ^Cluster cluster]
	  (let [needs-scheduling-topologies (.needsSchedulingTopologies cluster topologies)]
	    (doseq [^TopologyDetails topology needs-scheduling-topologies
            :let [topology-id (.getId topology)
                  available-slots (->> (.getAvailableSlots cluster)
                                       (map #(vector (.getNodeId %) (.getPort %))))
                  all-executors (->> topology
                                     .getExecutors
                                     (map #(vector (.getStartTask %) (.getEndTask %)))
                                     set)
                  alive-assigned (EvenScheduler/get-alive-assigned-node+port->executors cluster topology-id)
                  alive-executors (->> alive-assigned vals (apply concat) set);;获取存活的executor
                  can-reassign-slots (slots-can-reassign cluster (keys alive-assigned));;通过slots-can-reassign方法获得可以重新分配的slot
                  total-slots-to-use (min (.getNumWorkers topology)
                                          (+ (count can-reassign-slots) (count available-slots)));;获取Topology所能使用的全部slot
                  bad-slots (if (or (> total-slots-to-use (count alive-assigned));;如果预设的slot大于存活的slot 
                                    (not= alive-executors all-executors));;或预设的executor不等于存活的executor
                                (bad-slots alive-assigned (count all-executors) total-slots-to-use);;执行bad-slots方法获取bad-slot
                                [])]]
      (.freeSlots cluster bad-slots);;把bad-slot释放掉
      (EvenScheduler/schedule-topologies-evenly (Topologies. {topology-id topology}) cluster))));;执行EvenScheduler中的分配资源方法

如何获取可以被重新分配的slot:

	(defn slots-can-reassign [^Cluster cluster slots]
	  (->> slots
      (filter
        (fn [[node port]]
          (if-not (.isBlackListed cluster node);;如果slot所在的node不在黑名单中
            (if-let [supervisor (.getSupervisorById cluster node)]
              (.contains (.getAllPorts supervisor) (int port));;并且port在supervisor的port列表中
              ))))))

bad-slots方法如下：

	(defn- bad-slots [existing-slots num-executors num-workers];;参数分别为现存的slot，预设的executor，预设的slot
	  (if (= 0 num-workers)
    '()
    (let [distribution (atom (integer-divided num-executors num-workers));;integer-divided函数会把executor均匀的分配到worker上，并把预订方案赋值给distribution
          keepers (atom {})]
      (doseq [[node+port executor-list] existing-slots :let [executor-count (count executor-list)]]
        (when (pos? (get @distribution executor-count 0))
          (swap! keepers assoc node+port executor-list)
          (swap! distribution update-in [executor-count] dec)
          ));;这一步会把现存的existing-slots进行筛选，如果存在满足预订方案的现存部署，则从distribution中剔除，并放置到keepers中
      (->> @keepers
           keys
           (apply dissoc existing-slots);;从现存的existing-slots剔除keepers，剩下的就是不满足预订方案的部署，需要进行重新分配的
           keys
           (map (fn [[node port]]
                  (WorkerSlot. node port)))))))

##IsolationScheduler

这个Scheduler在初始化时就与前两者不同，他创建了一个container并赋值到保存在state中，并在调用prepare函数时把conf放到container中。

	(defn -init []
	  [[] (container)])

	(defn -prepare [this conf]
	  (container-set! (.state this) conf))

IsolationScheduler的schedule方法非常之长，让我们来一点一点进行分析：

	(defn -schedule [this ^Topologies topologies ^Cluster cluster]
	  (let [conf (container-get (.state this));;获取conf        
        orig-blacklist (HashSet. (.getBlacklistedHosts cluster));;获取blacklist
        iso-topologies (isolated-topologies conf (.getTopologies topologies));;从conf的ISOLATION-SCHEDULER-MACHINES中筛选出需要隔离的topology
        iso-ids-set (->> iso-topologies (map #(.getId ^TopologyDetails %)) set);;获取隔离的topology的id
        topology-worker-specs (topology-worker-specs iso-topologies);;获取隔离的topology对应的executor
        topology-machine-distribution (topology-machine-distribution conf iso-topologies);;的出machine和worker的最佳分配
        host-assignments (host-assignments cluster)];;获取当前集群中机器资源的分配情况
    (doseq [[host assignments] host-assignments]
      (let [top-id (-> assignments first second);;从分配情况中获取topologyid
            distribution (get topology-machine-distribution top-id);;获取理想情况下topology中machine和worker的最佳分配
            ^Set worker-specs (get topology-worker-specs top-id);;获取这个topology对应的executor信息
            num-workers (count assignments);;获取当前部署情况中的worker
            ]
        (if (and (contains? iso-ids-set top-id);;如果隔离topology包含这个topology
                 (every? #(= (second %) top-id) assignments);;并且当前部署的assignment都属于这个topology
                 (contains? distribution num-workers);;理想部署情况中包含当前部署的worker
                 (every? #(contains? worker-specs (nth % 2)) assignments));;executor也满足理想分配
          ;;这些条件都符合则进行一下操作：
          (do (decrement-distribution! distribution num-workers);;如果满足从理想部署情况，则把num-workers减1
              (doseq [[_ _ executors] assignments] (.remove worker-specs executors));;从worker-specs中把理想部署情况的executor去除
              (.blacklistHost cluster host));;把host加入blacklist，不会对这台机器做任何新任务的分配
          ;;如果不满足条件
          (doseq [[slot top-id _] assignments];;便利这台机器上的所有topology
            (when (contains? iso-ids-set top-id);;如果topology不属于隔离的
              (.freeSlot cluster slot);;释放这个slot
              ))
          )))
    
    (let [host->used-slots (host->used-slots cluster);;host对used-slot的映射
          ^LinkedList sorted-assignable-hosts (host-assignable-slots cluster)];;可用的host对slot的映射
      ;; TODO: can improve things further by ordering topologies in terms of who needs the least workers
      (doseq [[top-id worker-specs] topology-worker-specs
              :let [amts (distribution->sorted-amts (get topology-machine-distribution top-id))]];;从理想配置中选出隔离的topID对应的Worker并按降序排列
        (doseq [amt amts;;从worker中选择一个
                :let [[host host-slots] (.peek sorted-assignable-hosts)]];;从可用的host中查找第一个
          (when (and host-slots (>= (count host-slots) amt));;如果可用的host-slot存在，并且数量大于理想中topology需要的worker数
            (.poll sorted-assignable-hosts);;从可用的host中取出第一个
            (.freeSlots cluster (get host->used-slots host));;从cluster中释放这个host对应的slot
            (doseq [slot (take amt host-slots);;获取slot
                    :let [executors-set (remove-elem-from-set! worker-specs)]];;获取executor
              (.assign cluster slot top-id executors-set));;在cluster中设置topology对应的slot和executor
            (.blacklistHost cluster host));;黑名单中去掉这个host
          )))
    
    (let [failed-iso-topologies (->> topology-worker-specs
                                  (mapcat (fn [[top-id worker-specs]]
                                    (if-not (empty? worker-specs) [top-id])
                                    )))];;得到隔离的topology中没有被分配完的topology
      (if (empty? failed-iso-topologies);;如果未被分配的隔离top为空
        ;; run default scheduler on non-isolated topologies
        (-<> topology-worker-specs
             allocated-topologies
             (leftover-topologies topologies <>)
             (DefaultScheduler/default-schedule <> cluster));;所有topology中除去隔离的topology，其他的使用DefaultScheduler进行资源分配
        ;;如果有未被分配的隔离top为空
        (do
          (log-warn "Unable to isolate topologies " (pr-str failed-iso-topologies) ". No machine had enough worker slots to run the remaining workers for these topologies. Clearing all other resources and will wait for enough resources for isolated topologies before allocating any other resources.")
          ;; clear workers off all hosts that are not blacklisted
          (doseq [[host slots] (host->used-slots cluster)]
            (if-not (.isBlacklistedHost cluster host)
              (.freeSlots cluster slots)
              )));;释放所有不在blacklist中的host的slot，为下一次资源调度做准备
        ))
    (.setBlacklistedHosts cluster orig-blacklist);;把blacklist恢复到未被分配前的状态
    ))