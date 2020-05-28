
# Pending Pod数量
count(kube_pod_status_phase{namespace="ups-workflow", phase="Pending"} > 0) or vector(0)

# 动态流状态数
count(dynamicflow_status_phase > 0) by (phase)

# argo工作流状态数
count(argo_workflow_status_phase > 0) by (phase)

# 宿主机Pod数量监控
sum(kube_pod_info{namespace="ups-workflow",job="kubernetes-pods"}) by (node)

# 异常动态流状态监控
sum(dynamicflow_status_phase{namespace="ups-workflow", phase="Failed"} > 0) by (name, phase)

# 异常workflow状态监控
sum(argo_workflow_status_phase{phase=~"Failed|Error"} > 0) by (name, phase)

# 异常pod状态监控
count(kube_pod_status_phase{namespace="ups-workflow", phase!="Running",phase!="Succeeded"} > 0) by (pod, phase)

# 执行引擎存活监控
count(dynamicflow_status_phase) by (instance) * 0 + 1

# Argo workflow-controller存活监控
count(argo_workflow_info) by (instance) * 0 + 1

# argolog-operator存活监控
count(go_info{app="argolog-operator"}) by (instance) * 0 + 1

# ups-cronjob-controller存活监控
count(go_info{app="ups-cronjob"}) by (instance) * 0 + 1

# 动态流失败任务增长监控
idelta(dynamicflow_status_phase_failed[2m])

# Pushgateway存活监控
sum(pushgateway_build_info) by (instance)

# 各controller队列耗时分布
histogram_quantile(0.99, sum(rate(workqueue_work_duration_seconds_bucket[1m])) by (name, le))

# 各controller error reconcile数
sum(rate(controller_runtime_reconcile_errors_total[1m])) by (controller)
