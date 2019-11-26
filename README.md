# BPM Process Flow Perspective

This set of modules implements the standard structure to manage a flow of field value changes inside a module. In other words, given a module and a field, we will be able to define a set of validations and actions to be done when the field changes from one value to another.

The four modules are:

- **cbProcessFlow**: Defines the module and field to control along with a condition to filter which records in the module we want to apply the rules. It is related to a set of Steps that identify actions on each value change, a set of logs of the changes and a set of alerts or additional actions that can be made when the field spends a certain amount of time on one value.
- **cbProcessStep**: This module defines one exact value change, from one value to another. It forces a validation and a set of actions to execute if the validation is true and another set if the validation is false. It is also related to the process log module
- **cbProcessAlert**: This module defines a set of actions that must take place while the field being controlled is in a certain state.
- **ProcessLog**: is a simple logging module that can be used to register different events that occur during the process. This logging will not happen automatically. If you want to log events you will have to add the corresponding actions wherever you need them.

This looks like this:

![BPM Flow E-R](cbProcessFlow.png?raw=true "BPM Flow E-R")

## Functionality

When a record is saved we search the Process Flow module for records that control the module being saved, then we check if the controlled field is changing. If so, we evaluate the conditions on the Process Flow to see if the record is in the group of controlled records. If so, first, we see if there is a Dependency Graph related to the Flow. If there is we apply it's validation rules to decide if we will permit the change or not (see below). Next, we search the Process Step module for a step that represents the transition from the previous value of the field to the new value they are trying to save. if we find one, we apply the validations and then the positive or negative actions depending on the results. If no process step is found we proceed normally.

Every time the value of the field changes we save the transition information in an internal table which is scanned periodically with the Process Alert table settings in search of actions that must be launched in a timely manner.

## Dependency Graph Maps

![Dependency Graph](DependencyGraph.png?raw=true "Dependency Graph")

The Dependency Graph is the exact setting of the process you need to enforce. It will permit you to define the validations and actions to be taken when going from one value to the next (Process Step) and also the alerts and actions that need to be taken at a higher level of the process, like an overall restriction between two states.

This is a graphical representation of the Process Steps and Process Alerts records.

## Alerting Procedure

When a record is set to a value controlled by a Process Alert, a record is created in the alerting queue with the record ID, the alert ID and the next trigger time as defined by the schedule fields in the alert.

Every few minutes the queue is scanned in search of records whose trigger time has passed and those are launched:

- from the alert, we get the context map and evaluate with the record ID
- from the alert, we retrieve all the workflows and launch them with the context and record ID
- calculate next trigger time, if there is no next time we delete the record from the queue
- the record can also be deleted from the queue with the `deleteFromProcessAlertQueue` custom workflow method

## Push Along Developer Block

This developer block will show a graphical representation of the values the record can transition to depending on the current value assigned to the record and will permit you to move to one of the next steps by clicking on it.

## Future Enhancements

- Company workdays SLA considerations
- Create a set of records and workflows that implement some typical use cases like an **Approval Process** or **Force Potential Steps with Quote Versioning**.
