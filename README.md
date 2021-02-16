# kubernetes-cronhpa-controller 
[![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)
[![Build Status](https://travis-ci.org/AliyunContainerService/kubernetes-cronhpa-controller.svg?branch=master)](https://travis-ci.org/AliyunContainerService/kubernetes-cronhpa-controller)
## Overview 
`kubernetes-cronhpa-controller` is a kubernetes cron horizontal pod autoscaler controller using `crontab` like scheme. You can use `CronHorizontalPodAutoscaler` with any kind object defined in kubernetes which support `scale` subresource(such as `Deployment` and `StatefulSet`). 


## Installation 
1. install CRD 
```$xslt
kubectl apply -f config/crds/autoscaling_v1beta1_cronhorizontalpodautoscaler.yaml
```
2. install RBAC settings 
```$xslt
# create ClusterRole 
kubectl apply -f config/rbac/rbac_role.yaml

# create ClusterRolebinding and ServiceAccount 
kubectl apply -f config/rbac/rbac_role_binding.yaml
```
3. deploy kubernetes-cronhpa-controller
```$xslt
kubectl apply -f config/deploy/deploy.yaml
```
4. verify installation
```$xslt
kubectl get deploy kubernetes-cronhpa-controller -n kube-system -o wide 

➜  kubectl get deploy kubernetes-cronhpa-controller -n kube-system
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-cronhpa-controller   1         1         1            1           49s
```
## Example
Please try out the examples in the <a href="https://github.com/denis-jdsouza/kubernetes-cronhpa-controller/blob/master/examples">examples folder</a>.

1. Deploy sample workload and cronhpa
```$xslt
kubectl apply -f examples/deployment_cronhpa.yaml 
```

2. Check deployment replicas  
```$xslt
kubectl get deploy nginx-deployment-basic 

➜  kubectl get deploy nginx-deployment-basic
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment-basic   2         2         2            2           9s
```

3. Describe cronhpa status 
```$xslt
kubectl describe cronhpa cronhpa-sample

Name:         cronhpa-sample
Namespace:    default
Labels:       controller-tools.k8s.io=1.0
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"autoscaling.alibabacloud.com/v1beta1","kind":"CronHorizontalPodAutoscaler","metadata":{"annotations":{},"labels":{"controll...
API Version:  autoscaling.alibabacloud.com/v1beta1
Kind:         CronHorizontalPodAutoscaler
Metadata:
  Creation Timestamp:  2021-02-16T14:17:21Z
  Generation:          4
  Resource Version:    141099877
  Self Link:           /apis/autoscaling.alibabacloud.com/v1beta1/namespaces/default/cronhorizontalpodautoscalers/cronhpa-sample
  UID:                 bc053f72-64a9-45f6-8942-2ce3abd32087
Spec:
  Jobs:
    Discription:        for scale-up purpose
    Name:               scale-up-1
    Schedule:           30 */1 * * * *
    Target Percentage:  50
    Discription:        for scale-up purpose
    Name:               scale-up-2
    Schedule:           01 */1 * * * *
    Target Percentage:  30
  Scale Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         nginx-deployment-basic
Status:
  Conditions:
    Discription:        for scale-up purpose
    Job Id:             20665b8e-38a9-4706-a363-81fe8c5f42af
    Last Probe Time:    2021-02-16T14:17:30Z
    Message:            cron hpa job scale-up-1 executed successfully. current replicas:2, desired replicas:3, increment-by:1.
    Name:               scale-up-1
    Run Once:           false
    Schedule:           30 */1 * * * *
    State:              Succeed
    Target Percentage:  50
    Discription:        for scale-up purpose
    Job Id:             8bdbb48b-2126-4bb9-8055-ba5880a7fd5f
    Last Probe Time:    2021-02-16T14:18:01Z
    Message:            cron hpa job scale-up-2 executed successfully. current replicas:3, desired replicas:3, increment-by:0.
    Name:               scale-up-2
    Run Once:           false
    Schedule:           01 */1 * * * *
    State:              Succeed
    Target Percentage:  30
  Scale Target Ref:
    API Version:  apps/v1
    Kind:         Deployment
    Name:         nginx-deployment-basic
Events:
  Type    Reason   Age   From                         Message
  ----    ------   ----  ----                         -------
  Normal  Succeed  32s   CronHorizontalPodAutoscaler  cron hpa job scale-up-1 executed successfully. current replicas:2, desired replicas:3, increment-by:1.
  Normal  Succeed  1s    CronHorizontalPodAutoscaler  cron hpa job scale-up-2 executed successfully. current replicas:3, desired replicas:3, increment-by:0.
```

if the `State` of cronhpa job is `Succeed` that means the last execution is successful.
🍻Cheers! It works.

## Implementation Details
The following is an example of a `CronHorizontalPodAutoscaler`. 
```$xslt
apiVersion: autoscaling.alibabacloud.com/v1beta1
kind: CronHorizontalPodAutoscaler
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: cronhpa-sample
spec:
   scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-deployment-basic
   jobs:
   - name: "scale-up-1"
     discription: "for scale-up purpose" # optional
     schedule: "30 */1 * * * *"
     targetPercentage: 50  # will add 50% more replicas
   - name: "scale-up-2"
     discription: "for scale-up purpose" # optional
     schedule: "01 */1 * * * *"
     targetPercentage: 30  # will add 30% more replicas
``` 
The `scaleTargetRef` is the field to specify workload to scale. If the workload supports `scale` subresource(such as `Deployment` and `StatefulSet`), `CronHorizontalPodAutoscaler` should work well. `CronHorizontalPodAutoscaler` support multi cronhpa job in one spec. 

The cronhpa job spec need three fields:
* name    
  `name` should be unique in one cronhpa spec. You can distinguish different job execution status by job name.
* description    
  `description` description about the job, to make it easy for others to understand the purpose of job.
* schedule     
  The scheme of `schedule` is similar with `crontab`. `kubernetes-cronhpa-controller` use an enhanced cron golang lib （<a target="_blank" href="https://github.com/ringtail/go-cron">go-cron</a>） which support more expressive rules. 
  
  The cron expression format is as described below: 
  ```$xslt

    Field name   | Mandatory? | Allowed values  | Allowed special characters
    ----------   | ---------- | --------------  | --------------------------
    Seconds      | Yes        | 0-59            | * / , -
    Minutes      | Yes        | 0-59            | * / , -
    Hours        | Yes        | 0-23            | * / , -
    Day of month | Yes        | 1-31            | * / , - ?
    Month        | Yes        | 1-12 or JAN-DEC | * / , -
    Day of week  | Yes        | 0-6 or SUN-SAT  | * / , - ?    
  ```
  #### Asterisk ( * )    
  The asterisk indicates that the cron expression will match for all values of the field; e.g., using an asterisk in the 5th field (month) would indicate every month.
  #### Slash ( / )    
  Slashes are used to describe increments of ranges. For example 3-59/15 in the 1st field (minutes) would indicate the 3rd minute of the hour and every 15 minutes thereafter. The form "*\/..." is equivalent to the form "first-last/...", that is, an increment over the largest possible range of the field. The form "N/..." is accepted as meaning "N-MAX/...", that is, starting at N, use the increment until the end of that specific range. It does not wrap around.    
  #### Comma ( , )      
  Commas are used to separate items of a list. For example, using "MON,WED,FRI" in the 5th field (day of week) would mean Mondays, Wednesdays and Fridays.  
  #### Hyphen ( - )     
  Hyphens are used to define ranges. For example, 9-17 would indicate every hour between 9am and 5pm inclusive.   
  #### Question mark ( ? )      
  Question mark may be used instead of '*' for leaving either day-of-month or day-of-week blank.
  #### Predefined schedules
  You may use one of several pre-defined schedules in place of a cron expression.
  
  Entry                  | Description                                | Equivalent To
  -----                  | -----------                                | -------------
  @yearly (or @annually) | Run once a year, midnight, Jan. 1st        | 0 0 1 1 *
  @monthly               | Run once a month, midnight, first of month | 0 0 1 * *
  @weekly                | Run once a week, midnight between Sat/Sun  | 0 0 * * 0
  @daily (or @midnight)  | Run once a day, midnight                   | 0 0 * * *
  @hourly                | Run once an hour, beginning of hour        | 0 * * * *
  Intervals
  You may also schedule a job to execute at fixed intervals, starting at the time it's added or cron is run. This is supported by formatting the cron spec like this:
  
  @every <duration>
  where "duration" is a string accepted by time.ParseDuration (http://golang.org/pkg/time/#ParseDuration).
  
  For example, "@every 1h30m10s" would indicate a schedule that activates after 1 hour, 30 minutes, 10 seconds, and then every interval after that.
  
  Note: The interval does not take the job runtime into account. For example, if a job takes 3 minutes to run, and it is scheduled to run every 5 minutes, it will have only 2 minutes of idle time between each run.
  
  more schedule scheme please check this <a target="_blank" href="https://godoc.org/github.com/robfig/cron">doc</a>.
  #### Specific Date (@date)
  You may use the specific date to schedule a job for scaling the workloads. It is useful when you want to do a daily promotion.  
   
  Entry                       | Description                                | Equivalent To
  -----                       | -----------                                | -------------
  @date 2020-10-27 21:54:00   | Run once when the date reach               | 0 54 21 27 10 *
                              
* targetSize     
  `TargetPercentage` is the percentage of replics you desired to scale-up when the scheduled time arrive. 
  
* runOnce    
  if `runOnce` is true then the job will only run and exit after the first execution.
  
* excludeDates      
  excludeDates is a dates array. The job will skip the execution when the dates is matched. The minimum unit is day. If you want to skip the date(November 15th), You can specific the excludeDates like below.
  ```$xslt
    excludeDates:
    - "* * * 15 11 *"
  ```

## Common Question  
* Cloud `kubernetes-cronhpa-controller` and HPA work together?       
No
## Contributing
Please check <a href="https://github.com/AliyunContainerService/kubernetes-cronhpa-controller/blob/master/CONTRIBUTING.md">CONTRIBUTING.md</a>

## License
This software is released under the Apache 2.0 license.
