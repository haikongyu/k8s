# FATE组件介绍

## Reader

### 参数

```
table
	name: breast_hetero_guest
	namespace: experiment
```



### 功能

指明训练或测试模型所使用的数据

## DataIo

### 参数

```
outlier_replace_value: 0
outlier_replace_method: null
default_value: 0
label_type: int
delimitor: ,
input_format: dense
tag_value_delimitor: :
outlier_replace: false
missing_impute: null
output_format: dense
tag_with_value: false
data_type: float64
with_label: true
missing_fill_method: null
need_run: true
outlier_impute: null
label_name: y
exclusive_data_type: null
missing_fill: false
```

### 功能

读取数据到FATE供以下组件使用

## Intersection

### 参数

```
info_owner: guest
intersect_method: raw
encode_params
	salt:
	base64: false
	encode_method: none
intersect_cache_param
	encrypt_type: sha256
	id_type: phone
	use_cache: false
join_role: guest
only_output_key: false
random_bit: 128
allow_info_share: false
repeated_id_process: false
repeated_id_owner: guest
sync_intersect_ids: true
with_encode: false
```

### 功能

以安全的方式求出各方共有的实体

## HeteroFeatureBining

### 参数

```
bin_names: null
bin_num: 10
method: quantile
category_names: null
error: 0.001
transform_param
	transform_type: bin_num
	transform_cols: -1
	transform_names: null
category_indexes: null
optimal_binning_param
	metric_method: iv
	init_bin_nums: 1000
	mixture: true
	adjustment_factor: null
	max_bin: null
	init_bucket_method: quantile
	max_bin_pct: 1
	min_bin_pct: 0.05
skip_static: false
compress_thres: 10000
local_only: false
head_size: 10000
adjustment_factor: 0.5
need_run: true
bin_indexes: -1

```



### 功能

对特征进行分箱处理

## HeteroFeatureSelection

### 参数

```
unique_param
	eps: 0.00001
statistic_param
	filter_type: threshold
	select_federated: true
	threshold: 1
	metrics: mean
	host_thresholds: null
	take_high: true
manually_param
	filter_out_indexes: null
	left_col_indexes: null
	filter_out_names: null
	left_col_names: null
outlier_param
	percentile: 1
	upper_threshold: 1
select_names: []
iv_percentile_param
	local_only: false
	percentile_threshold: 1
sbt_param
	filter_type: threshold
	select_federated: true
	threshold: 1
	metrics: feature_importance
	host_thresholds: null
	take_high: true
iv_top_k_param
	local_only: false
	k: 10
filter_methods: [manually]
psi_param
	filter_type: threshold
	select_federated: true
	threshold: 1
	metrics: psi
	host_thresholds: null
	take_high: false
variance_coe_param
	value_threshold: 1
iv_param
	filter_type: threshold
	select_federated: true
	threshold: 1
	metrics: iv
	host_thresholds: null
	take_high: true
iv_value_param
	local_only: false
	value_threshold: 0
	host_thresholds: null
percentage_value_param
	upper_pct: 1
need_run: true
select_col_indexes: -1
```

### 功能

特征筛选

## HeteroLR

### 参数

```
predict_param
	threshold: 0.5
batch_size: 320
penalty: L2
early_stop: diff
early_stopping_rounds: null
validation_freqs: null
cv_param
	mode: hetero
	random_seed: 1
	n_splits: 5
	role: guest
	need_cv: false
	shuffle: true
decay: 1
decay_sqrt: true
tol: 0.0001
encrypted_mode_calculator_param
	mode: strict
	re_encrypted_rate: 1
optimizer: rmsprop
stepwise_param
	mode: hetero
	need_stepwise: false
	role: guest
	max_step: 10
	nvmax: null
	score_name: AIC
	direction: both
	nvmin: 2
alpha: 0.01
init_param
	random_seed: null
	fit_intercept: true
	init_method: random_uniform
	init_const: 1
multi_class: ovr
sqn_param
	random_seed: null
	sample_size: 5000
	update_interval_L: 3
	memory_M: 5
metrics: [auc, ks]
encrypt_param
	method: Paillier
	key_length: 1024
learning_rate: 0.15
max_iter: 3
use_first_metric_only: false
```

### 功能

## Evaluation

### 参数

```
allowed_metrics
	regression: [explained_variance, mean_absolute_error, mean_squared_error, median_absolute_error, r2_score, root_mean_squared_error]
	binary: [auc, ks, lift, gain, accuracy, precision, recall, roc, confusion_mat, psi, f1_score, quantile_pr]
	clustering: [jaccard_similarity_score, fowlkes_mallows_score, adjusted_rand_score, davies_bouldin_index, distance_measure, contingency_matrix]
	multi: [accuracy, precision, recall]
pos_label: 1
default_metrics
	regression: [explained_variance, mean_absolute_error, mean_squared_error, median_absolute_error, r2_score, root_mean_squared_error]
	binary: [auc, ks, lift, gain, accuracy, precision, recall, roc, confusion_mat, psi, f1_score, quantile_pr]
	clustering: [jaccard_similarity_score, fowlkes_mallows_score, adjusted_rand_score, davies_bouldin_index, distance_measure, contingency_matrix]
	multi: [accuracy, precision, recall]
metrics: null
need_run: true
eval_type: binary
run_clustering_arbiter_metric: false
```

### 功能



# 参考

## 在线表格生成网站

https://www.tablesgenerator.com/