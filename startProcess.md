@[toc]

# buildFlowDag
## dolphinscheduler的DAG流程
### 简要说明
此流程主要看org.apache.dolphinscheduler.server.master.runner.WorkflowExecuteRunnable#buildFlowDag

```
dolphinscheduler有很多_log 的表。之所以有这些表，笔者认为，当生成一个任务实例的时候，需要记录那个时刻的参数。
如果没有这些log 表，那么表中只会记录最新的记录。
如在某一个时刻生成了一个流程实例，此时该流程还在等待执行，当对流程进行修改，如果没有_log，则查出的参数为最新参数，而非生成流程实例的参数，所以我们需要_log 表。
```

### 代码分析

#### buildFlowDag 主流程
```java
private void buildFlowDag() throws Exception {
//根据版本号和流程定义code获取当前'流程实例'的'流程定义'记录
        processDefinition = processService.findProcessDefinition(processInstance.getProcessDefinitionCode(),
                processInstance.getProcessDefinitionVersion());
        processInstance.setProcessDefinition(processDefinition);
//找出cmdParam 中的StartNodeIdList（放的id）并根据此查询对应的TaskInstance
        List<TaskInstance> recoverNodeList = getRecoverTaskInstanceList(processInstance.getCommandParam());
//根据版本号和流程定义code获取 最新的ProcessTaskRelationLog
        List<ProcessTaskRelation> processTaskRelations =
                processService.findRelationByCode(processDefinition.getCode(), processDefinition.getVersion());
// 根据流程定义中的 pretask.和posttask 获取最新的任务定义
        List<TaskDefinitionLog> taskDefinitionLogs =
                processService.getTaskDefineLogListByRelation(processTaskRelations);
 //将获取当前taskDefinitionLogs 的本身信息 以及所有的pretaskCodes 
        List<TaskNode> taskNodeList = processService.transformTask(processTaskRelations, taskDefinitionLogs);
        forbiddenTaskMap.clear();
//如果taskExecuteType == TaskExecuteType.STREAM 则放入forbiddenTaskMap
        taskNodeList.forEach(taskNode -> {
            if (taskNode.isForbidden()) {
                forbiddenTaskMap.put(taskNode.getCode(), taskNode);
            }
        });

        // generate process to get DAG info
        List<String> recoveryNodeCodeList = getRecoveryNodeCodeList(recoverNodeList);
        List<String> startNodeNameList = parseStartNodeName(processInstance.getCommandParam());
        //将入参转换成DAG执行所需要的必要元素
        ProcessDag processDag = generateFlowDag(taskNodeList, startNodeNameList, recoveryNodeCodeList,
                processInstance.getTaskDependType());
        if (processDag == null) {
            logger.error("processDag is null");
            return;
        }
        //生成该processInstance的 DAG
        // generate process dag
        dag = DagHelper.buildDagGraph(processDag);
        logger.info("Build dag success, dag: {}", dag);
    }
```

#### 将入参转换成DAG执行所需要的必要元素
```java

//此方法我认为是做获取 所有的需要执行的任务节点 以及任务节点之间的关系

public static ProcessDag generateFlowDag(List<TaskNode> totalTaskNodeList,
                                             List<String> startNodeNameList,
                                             List<String> recoveryNodeCodeList,
                                             TaskDependType depNodeType) throws Exception {
//如果是 TaskDependType.TASK_POST startNodeIdList且不为null  又或者StartNodeList 不为null 则使用前端传递的参数。否则使用processInstace 关联的任务节点
        List<TaskNode> destTaskNodeList = generateFlowNodeListByStartNode(totalTaskNodeList, startNodeNameList,
                recoveryNodeCodeList, depNodeType);
        if (destTaskNodeList.isEmpty()) {
            return null;
        }
        //将destTaskNodeList每一个元素 解析成TaskNodeRelation。每一个元素都可能会有很多pretask。TaskNodeRelation.成员有startNode，endNode 分别代表 pretaskCode 和postTaskCode
        List<TaskNodeRelation> taskNodeRelations = generateRelationListByFlowNodes(destTaskNodeList);
        ProcessDag processDag = new ProcessDag();
        processDag.setEdges(taskNodeRelations);
        processDag.setNodes(destTaskNodeList);
        return processDag;
    }

```

#### DAG 参数概览
```java

package org.apache.dolphinscheduler.common.graph;


/**
 * analysis of DAG
 * Node: node
 * NodeInfo：node description information
 * EdgeInfo: edge description information
 */
public class DAG<Node, NodeInfo, EdgeInfo> {
    private static final Logger logger = LoggerFactory.getLogger(DAG.class);

    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    /**
     * node map, key is node, value is node information
     */
     //key 为taskCode value 是taskInstance的一些信息
    private final Map<Node, NodeInfo> nodesMap;

    /**
     * edge map. key is node of origin;value is Map with key for destination node and value for edge
     */
     //key startNode ,value 是所有依赖于startNode节点的map,该map key 依赖于startNode 的 taskCode，value 是该key 的具体信息
    private final Map<Node, Map<Node, EdgeInfo>> edgesMap;

    /**
     * reversed edge set，key is node of destination, value is Map with key for origin node and value for edge
     */
     //key 是 endNode 的 taskCode ,map是endNode 依赖的所有节点
    private final Map<Node, Map<Node, EdgeInfo>> reverseEdgesMap;

    public DAG() {
        nodesMap = new HashMap<>();
        edgesMap = new HashMap<>();
        reverseEdgesMap = new HashMap<>();
    }

}


```
#### DAG生成逻辑
```java 

 public static DAG<String, TaskNode, TaskNodeRelation> buildDagGraph(ProcessDag processDag) {

        DAG<String, TaskNode, TaskNodeRelation> dag = new DAG<>();
//添加DAG中的nodesMap
        // add vertex
        if (CollectionUtils.isNotEmpty(processDag.getNodes())) {
            for (TaskNode node : processDag.getNodes()) {
                dag.addNode(Long.toString(node.getCode()), node);
            }
        }

        // add edge
        if (CollectionUtils.isNotEmpty(processDag.getEdges())) {
            for (TaskNodeRelation edge : processDag.getEdges()) {
            //添加DAG中的edgesMap 与reverseEdgesMap
                dag.addEdge(edge.getStartNode(), edge.getEndNode());
            }
        }
        return dag;
    }
```

##### 生成DAG中的edgesMap 与reverseEdgesMap
```java

 public boolean addEdge(Node fromNode, Node toNode, EdgeInfo edge, boolean createNode) {
        lock.writeLock().lock();

        try {
            // Whether an edge can be successfully added(fromNode -> toNode)
            //1.如果fromNode 指向了 toNode 2.如果DAG的nodeMap 里面不包含fromNode 或者toNode 3.如果toNode的下游节点或者下下游，以此类推的节点指向了fromNode节点。则返回false 报错
            if (!isLegalAddEdge(fromNode, toNode, createNode)) {
                logger.error("serious error: add edge({} -> {}) is invalid, cause cycle！", fromNode, toNode);
                return false;
            }

            addNodeIfAbsent(fromNode, null);
            addNodeIfAbsent(toNode, null);

            addEdge(fromNode, toNode, edge, edgesMap);
            addEdge(toNode, fromNode, edge, reverseEdgesMap);

            return true;
        } finally {
            lock.writeLock().unlock();
        }

    }

```
至此buildDAG 的方法解析结束，具体看DAG里包含的三个参数



# initTaskQueue

### 简要说明
-  疑问
```
笔者刚看到这里感觉很奇怪，因为当processInstance第一次执行的时候，如果不是ComplementData 类型的。initTaskQueue 方法基本上只执行clear方法。为什么写那么多
```
- 解答
```
在api服务中执行任务流程生成任务实例时，是生成command插入表中，在master中将command转换成processInstance,此时执行状态可能是WorkflowExecutionStatus.SERIAL_WAIT,此时就不是RUNNING_EXECUTION 状态，可能是专门为这种类型准备的。
```

#### initTaskQueue 主流程代码
```java

 private void initTaskQueue() {

        taskFailedSubmit = false;
        activeTaskProcessorMaps.clear();
        dependFailedTaskMap.clear();
        completeTaskMap.clear();
        errorTaskMap.clear();
//1.如果该流程实例重新恢复的返回false 2.如果任务流程实例是RUNNING_EXECUTION 状态，并且是第一次运行则返回true 3.其他则返回false
        if (!isNewProcessInstance()) {
        //如果不是新的流程实例了，找到taskInstance表中，process_instance_id =  processInstance.getId()并且flag=1 的（1为可获得）

            List<TaskInstance> validTaskInstanceList = processService.findValidTaskListByProcessId(processInstance.getId());
            for (TaskInstance task : validTaskInstanceList) {
            //将validTaskMap缓存中相同code的任务实例替换
                if (validTaskMap.containsKey(task.getTaskCode())) {
                    int oldTaskInstanceId = validTaskMap.get(task.getTaskCode());
                    TaskInstance oldTaskInstance = taskInstanceMap.get(oldTaskInstanceId);
                    if (!oldTaskInstance.getState().typeIsFinished() && task.getState().typeIsFinished()) {
                    //如果结束了就设置成不可获取，并更新到数据库中
                        task.setFlag(Flag.NO);
                        processService.updateTaskInstance(task);
                        continue;
                    }
                    logger.warn("have same taskCode taskInstance when init task queue, taskCode:{}", task.getTaskCode());
                }

                validTaskMap.put(task.getTaskCode(), task.getId());
                taskInstanceMap.put(task.getId(), task);
				//如果该任务完成了就放入完成任务的缓存中
                if (task.isTaskComplete()) {
                    completeTaskMap.put(task.getTaskCode(), task.getId());
                    continue;
                }
                //如果该任务是conditions 节点类型的任务 或者后续有conditions节点类型的任务则跳过
                if (task.isConditionsTask() || DagHelper.haveConditionsAfterNode(Long.toString(task.getTaskCode()), dag)) {
                    continue;
                }
                if (task.taskCanRetry()) {
                    if (task.getState() == ExecutionStatus.NEED_FAULT_TOLERANCE) {
                        // tolerantTaskInstance add to standby list directly
                        TaskInstance tolerantTaskInstance = cloneTolerantTaskInstance(task);
                        addTaskToStandByList(tolerantTaskInstance);
                    } else {
                        retryTaskInstance(task);
                    }
                    continue;
                }
                //如果失败则放入失败任务缓存中
                if (task.getState().typeIsFailure()) {
                    errorTaskMap.put(task.getTaskCode(), task.getId());
                }
            }
        }
		//如果processInstance的历史cmd的commandType 是CommandType.COMPLEMENT_DATA类型的 并且complementListDate大小是0
        if (processInstance.isComplementData() && complementListDate.size() == 0) {
            Map<String, String> cmdParam = JSONUtils.toMap(processInstance.getCommandParam());
            if (cmdParam != null && cmdParam.containsKey(CMDPARAM_COMPLEMENT_DATA_START_DATE)) {
                // reset global params while there are start parameters
                setGlobalParamIfCommanded(processDefinition, cmdParam);

                Date start = DateUtils.stringToDate(cmdParam.get(CMDPARAM_COMPLEMENT_DATA_START_DATE));
                Date end = DateUtils.stringToDate(cmdParam.get(CMDPARAM_COMPLEMENT_DATA_END_DATE));
                List<Schedule> schedules = processService.queryReleaseSchedulerListByProcessDefinitionCode(processInstance.getProcessDefinitionCode());
                if (complementListDate.size() == 0 && needComplementProcess()) {
                    complementListDate = CronUtils.getSelfFireDateList(start, end, schedules);
                    logger.info(" process definition code:{} complement data: {}",
                        processInstance.getProcessDefinitionCode(), complementListDate.toString());

                    if (complementListDate.size() > 0 && Flag.NO == processInstance.getIsSubProcess()) {
                        processInstance.setScheduleTime(complementListDate.get(0));
                        processInstance.setGlobalParams(ParameterUtils.curingGlobalParams(
                            processDefinition.getGlobalParamMap(),
                            processDefinition.getGlobalParamList(),
                            CommandType.COMPLEMENT_DATA, processInstance.getScheduleTime(), cmdParam.get(Constants.SCHEDULE_TIMEZONE)));
                        processService.updateProcessInstance(processInstance);
                    }
                }
            }
        }
    }


```

