# Advanced concepts

[SubDAGs](#subdags)  
[TaskGroups](#taskgroups)  
[XComs](#xcoms)  
[Branching](#branching)  
[Trigger Rules](#trigger-rules)  

## SubDAGs

SubDAGs allow us to create a DAG inside another DAG so that we can group similar tasks together. To use SubDAGs we need the `SubDAG` operator. Then we need to create a function who's return value is the SubDAG and use that function as the value of the `subdag` argument of the operator. These SubDAGs should be placed on the `airflow/dags/subdags` directory.

The subdag function takes three arguments:

- `parent_dag_id`: the ID of the parent DAG
- `child_dag_id`: the ID of the child DAG
- `default_args`: the same dictionary of default arguments as the parent DAG. Especially important is that the SubDAG has the same `start_date` and the same schedule interval.

SubDAGs are not recommended to be used due to:

- can lead to deadlocks preventing other tasks from being run
- high complexity of implementation
- SubDAGs have their own Sequential Executor so parallelism can be leveraged

## TaskGroups

Task groups are much easier than SubDAGs to build. We don't need a function that returns a DAG. Just invoke the `TaskGroup` operator. Give the task group an ID as an argument to the group, and include the tasks as part of the object.

We can use sub tasks in task groups. Tasks under different groups can have the same ID, since their actual ID will be `group_task_name.task_id`.

## XComs

XComs are used to share data between tasks in Airflow. XComs stands for *cross communication* and allows for the exchange of small pieces of data between tasks. The data that is being shared is stored in the Airflow Metadata base. Therefore, te amount of data that can be stored depends on the DB used. For SQLite it's limited to 2GB. For PostgreSQL it's 1GB. For MySQL the limit is 64KB.

The XCom is pushed onto the DB by passing it as the return value of the task that generates it. The next task will then pull that value. If we want to push the XCom with a specific key, we need to use the `xcom_push` method and specify the key name and the value. This method is part of the task instance, and thus, the task instance needs to be passed as an argument (`ti`) to the function.

``` python
def _training_model(ti):
  accuracy = uniform(0.1, 10.0)
  return ti.xcom_push(key = 'model_accuracy', value = accuracy)
```

If we push two different XComs with the same key but different values, the last one pushed will overwrite the first one pushed.

Now, to pull the value by a task we use the `xcom_pull` method of the `ti`. This method takes two arguments. The `key` is the name of the XCom to be pulled, and a list of `task_ids` which XComs we want to pull.

Some operators will create XComs by default (such as the `BashOperator`). To modify that behavior, change the value of the parameter `do_xcom_push` to `False`.

## Branching

Sometimes we need to choose one task or another depending on the value of an XCom. To do so, we use the `BranchPythonOperator`. This operator allows us to execute one task or another by returning the task ID of the task to be executed.

We could return multiple task IDs from the `BranchPythonOperator`. To do so, just specify the return value as a list. All tasks in that list will be executed.

## Trigger rules

We can also want to execute other tasks after the branched step in our pipeline. To do so, we need to change the Trigger Rules. By default, all task use the trigger rule `all_success` which means that all previous tasks must have been executed successfully for the following task in the data pipeline to be executed. Similarly, the `all_failed` trigger rule will execute the task if all dependent tasks have failed.

Other rules include:

- `all_done` allows for the execution of downstream tasks, whatever the status of the upstream task.
- `one_success` will trigger the downstream task as soon as at least one upstream task succeeds.
- `one_failed` will trigger the downstream task as soon as at least one upstream task fails.
- `none_failed` will trigger the downstream task as soon as all upstream tasks have succeeded or have been skipped.
- `none_failed_or_skipped` will trigger the downstream task as soon as all upstream tasks have not failed, but at least one upstream task has succeeded.
