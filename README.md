# BPM Process Flow Perspective

This set of modules implements the standard structure to manage a flow of field value changes inside a module. In other words, given a module and a field, we will be able to define a set of validations and actions to be done when the field changes from one value to another.

The four modules are:

- **cbProcessFlow**: Defines the module and field to control along with a condition to filter which records in the module we want to apply the rules. It is related to a set of Steps that identify actions on each value change, a set of logs of the changes, and a set of alerts or additional actions that can be made when the field spends a certain amount of time on one value.
- **cbProcessStep**: This module defines one exact value change, from one value to another. It forces a validation and a set of actions to execute if the validation is true and another set if the validation is false. It is also related to the process log module
- **cbProcessAlert**: This module defines a set of actions that must take place while the field being controlled is in a certain state.
- **ProcessLog**: is a simple logging module that can be used to register different events that occur during the process. This logging will not happen automatically. If you want to log events you will have to add the corresponding actions wherever you need them.

This looks like this:

![BPM Flow E-R](cbProcessFlow.png?raw=true "BPM Flow E-R")

## Functionality

When a record is saved we search the Process Flow module for records that control the module being saved, then we check if the controlled field is changing. If so, we evaluate the conditions on the Process Flow to see if the record is in the group of controlled records. If so, we search the Process Step module for a step that represents the transition from the previous value of the field to the new value they are trying to save. if we find one, we apply the validations and then the positive or negative actions depending on the results. If no process step is found we proceed normally (this shouldn't happen really, you should define a step for each valid transition).

The actions above happen in a Validation map created when the Process Flow record is created. That validation executes the code in `modules/cbProcessFlow/validateFlowStep.php`. If the validation passes then the aftersave event is triggered to put the workflows in the queue. This happens in `modules/cbProcessAlert/AlertSettingsHandler.php`. The workflows on the queue are processed by the `modules/cbProcessAlert/cron/alertqueue.php` script.

A step can be marked as inactive but each step also has an "Is Active Validation" map. If this field contains a validation map that evaluates to "false" then the step will not be applied. For the validation to be evaluated the step must be marked as active.

There is a special case that is so common that we have a simplified feature for it. In most BPM processes, each step will change the assigned user of the record. You can do this with the BPM system as it is. You create a workflow that has an update field task. Done this way, you have the possibility of executing the assignment in any sequence you may need. For example, if you need to do two tasks before changing the assigned user and two more after, you can do this with the workflow. But, many times, we just need to change the assigned user and, maybe, some other additional workflow tasks. For this particular use case, we have added two fields to the step. A business map reference field and a checkbox name "Assign User/Group After". The functionality is that if the map exists it will be executed to get a User or Group ID that will be used to change the assigned user of the record. The map may be a condition expression, a condition query, or a decision table. If there are other workflows related to the step we can indicate if we want to execute the assignment BEFORE or AFTER those workflows using the checkbox field. Notice that there is no option to execute the assignment in between the workflows, if you need to do that you must use an update field workflow task in the sequence you require.

Every time the value of the field changes we save the transition information in an internal table which is scanned periodically with the Process Alert table settings in search of actions that must be launched in a timely manner.

[Video Presentation](https://youtu.be/QOKuNtXGls4)


## Dependency Graph Maps

![Dependency Graph](DependencyGraph.png?raw=true "Dependency Graph")

The Dependency Graph is the exact setting of the process you need to enforce. It will permit you to define the validations and actions to be taken when going from one value to the next (Process Step) and also the alerts and actions that need to be taken at a higher level of the process, like an overall restriction between two states.

This is a graphical representation of the Process Steps and Process Alerts records.

This is not used in the project, it is just for the implementor to get a view of the layout they have constructed.


## Alerting Procedure

When a record is set to a value controlled by a Process Alert, a record is created in the alerting queue with the record ID, the alert ID, and the next trigger time as defined by the schedule fields in the alert.

Every few minutes the queue is scanned in search of records whose trigger time has passed and those are launched:

- from the alert, we get the context map and evaluate it with the record ID
- from the alert, we retrieve all the workflows and launch them with the context and record ID
- calculate the next trigger time, if there is no next time we delete the record from the queue
- the record can also be deleted from the queue with the `deleteFromProcessAlertQueue` custom workflow method

## Push Along Developer Block

This developer block will show a graphical representation of the values the record can transition to depending on the current value assigned to the record and will permit you to move to one of the next steps by clicking on it.

The default presentation is a mermaid graph showing the current state and the possible next states the user can go to. The label used for the state will be the state itself unless you specify a different value in the `Button Label` field.

Alternatively, the Push Along block can show the next states as a stack of buttons. In this presentation, the current state is not shown, only the available destination states. By default, the label of the button will be the state but you can set any value using the `Button Label` field. In the case of presenting buttons, the `Button Label` field supports an enhanced syntax that permits us to define the class to use.

`{"label":"some label", "type":"approve|decline|css_class"}`

If only the type is given, the destination state value will be used.

To change the mermaid view to the button view you must edit the business action which is created by default and add the `showas=buttons` parameter.

`module=cbProcessFlow&action=cbProcessFlowAjax&file=pushAlongFlow&id=$RECORD$&pflowid=44964&showas=buttons`

## Workflow Context Variables

Both the step and alert process set some context variables that are passed along the workflow execution chain so they can be used in the tasks. For example, when creating a Process Log record, we are normally in the context of the record launching the process but we need to get information about the Process Step or Alert that launched the workflow. Let's suppose we have a process that is controlling the status changes of a project task. When a task changes its status a Process Step launches a workflow that executes a create record task to create a Process Log (we want to log the status change). The creation is launched from the Project Task and we have no idea of, nor any way to get information from, the Process Step that started everything. To permit you to access that information the Process Flow system loads some variables in the workflow context so you can retrieve those values and use them in workflow tasks with the `getFromContext` method.

These variables are:

- **ProcessRelatedFlow**: crmid of the process flow controlling the field value change
- **ProcessRelatedStepOrAlert**: crmid of the step or alert launching the workflow
- **ProcessPreviousStatus**: previous value of the field if we are being launched from a Step or empty if it is an Alert launching the workflow task
- **ProcessNextStatus**: next status that has been saved on the record or that would have been saved if the Step validations did not pass (negative workflows) or the Alert status

![Create Entity with Context](CreateEntityWithContext.png?raw=true "Create Entity with Context")

Additionally, the `deleteFromProcessAlertQueue` will look in the context for a variable named: `ProcessAlertValueToDelete`. If this variable (which can be loaded through the context map in the Alert) has a valid CRMID of an Alert that Alert will be eliminated from the queue.

## Installing

Either use the `composer.json` file and composer or copy the changeset (processflow.xml) to the coreBOS updater cbupdates directory, copy all the modules into their place and use coreBOS Updater.

## Future Enhancements

- send step validation errors to top-level UI
- Company workdays SLA considerations
- Create a set of records and workflows that implement some typical use cases like an **Approval Process** or **Force Potential Steps with Quote Versioning**.
