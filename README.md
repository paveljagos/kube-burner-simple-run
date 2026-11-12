# kube-burner first rest run



* [install kubeburner](https://github.com/kube-burner/kube-burner)
```sh
curl -Ls https://raw.githubusercontent.com/kube-burner/kube-burner/refs/heads/main/hack/install.sh | sh
```


## run tests
```sh
export KUBECONFIG=/home/$USER/install-out/auth/kubeconfig

# In-cluster Prometheus (works from outside the cluster via the route).
export PROMETHEUS_URL="https://$(oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}')"
export PROMETHEUS_TOKEN="$(oc create token prometheus-k8s -n openshift-monitoring --duration=6h)"


cd ./tests
kube-burner health-check
time="2026-11-02 07:26:14" level=info msg="🏥 Checking for Cluster Health" file="cluster_health.go:14"

# Run a small cluster-density workload
time kube-burner init -c kubelet-density-crio.yml
```

### see the metrics
```sh
$ tar -tzvf new-kubelet-density-metrics.tgz   
-rw-r--r-- lab-user/users 125153 2026-11-04 07:04 API99thLatency.json
-rw-r--r-- lab-user/users   1313 2026-11-04 07:04 APIInflightRequests.json
-rw-r--r-- lab-user/users  95291 2026-11-04 07:04 APIRequestRate.json
-rw-r--r-- lab-user/users    671 2026-11-04 07:04 jobSummary.json
```




## duckdb: get data in

The simple analysis of data follows using `duckdb` and `R`.

As testing environment i was using [single node openshift in KVM](https://github.com/paveljagos/sno-in-kvm) where i was able to change `number of vCPUs` and get *metrics*.


### latency data from tests on cpu 24 34 44 54

* load the data into the duckdb
```sql
CREATE TABLE vcpu24 AS SELECT * FROM read_json('wp-24cpu/API99thLatency.json');
CREATE TABLE vcpu34 AS SELECT * FROM read_json('wp-34cpu/API99thLatency.json');
CREATE TABLE vcpu44 AS SELECT * FROM read_json('wp-44cpu/API99thLatency.json');
CREATE TABLE vcpu54 AS SELECT * FROM read_json('wp-54cpu/API99thLatency.json');
```


* create table with all latency values
```sql
CREATE TABLE allvalues AS                                                            
   SELECT n1.value AS vcpu24,                                                           
          n2.value AS vcpu34,                                                           
          n3.value AS vcpu44,                                                           
          n4.value AS vcpu54                                                            
   FROM (SELECT row_number() OVER () AS rn, value FROM vcpu24) n1                       
   FULL JOIN (SELECT row_number() OVER () AS rn, value FROM vcpu34) n2 USING (rn)       
   FULL JOIN (SELECT row_number() OVER () AS rn, value FROM vcpu44) n3 USING (rn)       
   FULL JOIN (SELECT row_number() OVER () AS rn, value FROM vcpu54) n4 USING (rn)       
   ORDER BY rn;
```

* save  to file
```sql
ATTACH 'vcpu24-34-44-54.duckdb' AS allcpu;
COPY FROM DATABASE memory TO allcpu;
DETACH allcpu;
```


> first look at data in duckdb ui:
```sh
duckdb API99thLatency.duckdb -ui
```


#### analysis R

> null hypotesis: there is no significat difference between measurements (values)

##### R: H0 confirmed, no significant difference


* install and load packages 
```sh
install.packages("duckdb")   
install.packages("tidyr")
install.packages("dplyr")
install.packages("dbplyr")
install.packages("ggplot2")

library(ggplot2)   
library(duckdb)
library(tidyr)
library(dplyr)
library(dbplyr)
```


* load data from `.duckdb` [duckdb](https://cran.r-project.org/web/packages/duckdb/duckdb.pdf)
```R
con <- dbConnect(duckdb(), dbdir = "./vcpu24-34-44-54.duckdb")

rawdata <- tbl(con, "allvalues")

cpudata <- pivot_longer(    # https://cran.r-project.org/web/packages/tidyr/tidyr.pdf
  rawdata, 
  cols = everything(), 
  names_to = "ncpu_as_group", 
  values_to = "latency"
)

head(cpudata)


# https://cambiotraining.github.io/stats-week/materials/cs2_practical_anova.html
lm_latency <- lm(latency ~ ncpu_as_group, data = cpudata)
anova(lm_latency)


###############
# One-way ANOVA  -  this is build in: https://cran.r-project.org/web/packages/HSAUR/vignettes/Ch_analysis_of_variance.pdf
model <- aov(latency ~ ncpu_as_group, data = cpudata)
summary(model)   


boxplot(latency ~ ncpu_as_group, data = cpudata)
ggplot(cpudata, aes(x = ncpu_as_group, y = latency)) +  geom_boxplot()
```





#### uplot

This is for fun, cause i discover the [youplot](https://github.com/red-data-tools/YouPlot).

```sh
$ duckdb vcpu24-34-44-54.duckdb -noheader -separator $'\t' -s "
  UNPIVOT (
    SELECT SUM(COLUMNS(*)) FROM allvalues
  ) ON COLUMNS(*) INTO NAME vcpu VALUE total;
"
┌─────────┬────────────────────┐
│  vcpu   │       total        │
│ varchar │       double       │
├─────────┼────────────────────┤
│ vcpu24  │ 5.0954558635136795 │
│ vcpu34  │ 2.6917692779844105 │
│ vcpu44  │ 3.0050386662688315 │
│ vcpu54  │  4.749216030605943 │
└─────────┴────────────────────┘




$ duckdb vcpu24-34-44-54.duckdb -noheader -csv -s "
  UNPIVOT (
    SELECT SUM(COLUMNS('vcpu.*')) FROM allvalues
  ) ON COLUMNS(*) INTO NAME vcpu VALUE total;
" | uplot bar -d ',' -t "Total Sum by vCPU"
                      Total Sum by vCPU
          ┌                                        ┐ 
   vcpu24 ┤■■■■■■■■■■■■■■■■■■■■ 5.0954558635136795   
   vcpu34 ┤■■■■■■■■■■■ 2.6917692779844105            
   vcpu44 ┤■■■■■■■■■■■■ 3.0050386662688315           
   vcpu54 ┤■■■■■■■■■■■■■■■■■■■ 4.749216030605943     
          └                                        ┘ 
```

