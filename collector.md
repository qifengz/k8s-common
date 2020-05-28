package metrics

import (
	"github.com/prometheus/client_golang/prometheus"
	upsv1alpha1 "gitlab.alibaba-inc.com/uc-ups/execution-engine/pkg/api/v1alpha1"
	"gitlab.alibaba-inc.com/uc-ups/execution-engine/pkg/common"
	"sigs.k8s.io/controller-runtime/pkg/metrics"
	"strings"
	"sync"
)

var (
	dynamicflowStatusPhase = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "dynamicflow_status_phase",
			Help: "The dynamicflow current phase.",
		},
		[]string{"namespace", "name", "phase"},
	)

	dynamicflowInfo = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "dynamicflow_info",
			Help: "The information of dynamicflow.",
		},
		[]string{"namespace", "name", labelKeyToPromLabel(common.LabelKeyDynamicflowSceneID), labelKeyToPromLabel(common.LabelKeyDynamicflowTimeoutAlarmEnable), labelKeyToPromLabel(common.LabelKeyDynamicflowTimeoutSecond), common.PrometheusLabelDynamicflow},
	)
	dfInfoMap sync.Map

	dynamicflowStartedAt = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name: "dynamicflow_start_time",
			Help: "Start time in unix timestamp for a dynamicflow.",
		},
		[]string{"namespace", "name"},
	)

	//dynamicflowFinishedAt = prometheus.NewGaugeVec(
	//	prometheus.GaugeOpts{
	//		Name: "dynamicflow_completion_time",
	//		Help: "Completion time in unix timestamp for a dynamicflow.",
	//	},
	//	[]string{"namespace", "name", labelKeyToPromLabel(common.LabelKeyDynamicflowTimeoutAlarmEnable), labelKeyToPromLabel(common.LabelKeyDynamicflowTimeoutSecond)},
	//)
)

func getMetricCacheKey(namespace string, name string) string {
	return namespace + "`" + name
}

func boolFloat64(b bool) float64 {
	if b {
		return 1
	}
	return 0
}

func CollectDynamicflow(df *upsv1alpha1.Dynamicflow) {
	addGauge := func(g *prometheus.GaugeVec, v float64, lv ...string) {
		lv = append([]string{df.Namespace, df.Name}, lv...)
		m := g.WithLabelValues(lv...)
		m.Set(v)
	}
	deleteGauge := func(g *prometheus.GaugeVec, lv ...string) {
		lv = append([]string{df.Namespace, df.Name}, lv...)
		g.DeleteLabelValues(lv...)
	}

	addGauge(dynamicflowStatusPhase, boolFloat64(df.Status.Phase == upsv1alpha1.DynamicFlowInit || df.Status.Phase == ""), upsv1alpha1.DynamicFlowInit)
	addGauge(dynamicflowStatusPhase, boolFloat64(df.Status.Phase == upsv1alpha1.DynamicFlowRunning), upsv1alpha1.DynamicFlowRunning)
	addGauge(dynamicflowStatusPhase, boolFloat64(df.Status.Phase == upsv1alpha1.DynamicFlowFailed), upsv1alpha1.DynamicFlowFailed)
	addGauge(dynamicflowStatusPhase, boolFloat64(df.Status.Phase == upsv1alpha1.DynamicFlowSucceeded), upsv1alpha1.DynamicFlowSucceeded)

	if !df.Status.StartedAt.IsZero() {
		addGauge(dynamicflowStartedAt, float64(df.Status.StartedAt.Unix()))
	}

	sceneID, _ := df.Labels[common.LabelKeyDynamicflowSceneID]
	enable := df.Labels[common.LabelKeyDynamicflowTimeoutAlarmEnable]
	timeoutSecond := df.Labels[common.LabelKeyDynamicflowTimeoutSecond]
	var completed string
	if df.Status.Completed() {
		completed = "true"
	} else {
		completed = "false"
	}
	//清理旧metric
	labelList, ok := dfInfoMap.Load(getMetricCacheKey(df.Namespace, df.Name))
	if ok {
		labelList := labelList.([]string)
		deleteGauge(dynamicflowInfo, labelList...)
	}
	dfInfo := []string{sceneID, enable, timeoutSecond, completed}
	addGauge(dynamicflowInfo, 1, dfInfo...)
	dfInfoMap.Store(getMetricCacheKey(df.Namespace, df.Name), dfInfo)
}

func DeleteDynamicflowMetric(namespace string, name string) {
	deleteGauge := func(g *prometheus.GaugeVec, lv ...string) {
		lv = append([]string{namespace, name}, lv...)
		g.DeleteLabelValues(lv...)
	}
	deleteGauge(dynamicflowStatusPhase, upsv1alpha1.DynamicFlowInit)
	deleteGauge(dynamicflowStatusPhase, upsv1alpha1.DynamicFlowRunning)
	deleteGauge(dynamicflowStatusPhase, upsv1alpha1.DynamicFlowFailed)
	deleteGauge(dynamicflowStatusPhase, upsv1alpha1.DynamicFlowSucceeded)

	deleteGauge(dynamicflowStartedAt)

	labelList, ok := dfInfoMap.Load(getMetricCacheKey(namespace, name))
	if ok {
		labelList := labelList.([]string)
		deleteGauge(dynamicflowInfo, labelList...)
		dfInfoMap.Delete(getMetricCacheKey(namespace, name))
	}
}

func labelKeyToPromLabel(key string) string {
	rplKey := strings.ReplaceAll(key, ".", "_")
	return "label_" + rplKey
}

func init() {
	metrics.Registry.MustRegister(dynamicflowStatusPhase, dynamicflowInfo, dynamicflowStartedAt)
}