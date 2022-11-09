- Start Date: 23.03.2022
- Status: Built

# Using Plans and Activities through renku.api

## Summary

Add support for accessing Activities and Plans in the current project, along
with their child properties.

## Motivation

As users use more workflow features and more complex workflow pipelines, a way
for programatically seeing workflows in the current project becomes more
important.
Reading Activities and Plans allows meta-analysis in repositories, such as
summarizing over multiple executions of the same Plan.
Additionally, it allows writing scripts that detect existing workflows and
to then shell out to `renku workflow execute` to run Plans dynamically.


## Design Detail

Add new classes to `renku.api`:

- Activity
- UsedInput
- CreatedOutput
- FieldValue
- Plan
- CompositePlan
- Input
- Output
- Parameter
- Mapping
- Link

The classes wrap core renku classes and expose a simplified subset of
properties.

### Class Diagram
This diagram shows each class and the properties/methods that the user can use to navigate between the objects, as indicated by the arrows. Dashed lines indicate logical relationships between classes that are not made available to the user.

_Note:_ The methods from `Mapping` and `Link` have been excluded from this diagram for the sake of readability, though they are included as comments in the diagram's code.

```mermaid
classDiagram
    class Project{
        +path path
        +Client client
        +ProjectStatus status()
    }
    class Dataset{
        +List-DatasetFile files
        +List-Dataset list()
    }
    class DatasetFile{
        +datetime date_added
        +string name
        +path path
        +unknown entity
    }
    class ProjectStatus{
        +List-string             stale_outputs
        +List-renku-api-Activity stale_activities
        +List-string             modified_inputs
        +List-string             deleted_inputs
    }
    class Plan{
        +string name
        +string description
        +datetime date_created
        +List-string keywords
        +List-renku_api_Input inputs
        +List-renku_api_Output outputs
        +List-renku_api_Parameter parameters
        +string command
        +List-string keywords
        +List-int success_codes
        +bool deleted
        +List-renku_api_Activity activities
        +List-renku_api_Plan list()

    }
    class Input {
      +string name
      +string description
      +string value
      +string prefix
      +int    position
      +string mapped_stream
    }
    class Output {
      +string name
      +string description
      +string value
      +string prefix
      +int    position
      +string mapped_stream
    }
    class Parameter {
      +string name
      +string description
      +string value
      +string prefix
      +int    position
    }
    class CompositePlan {
        +string  name
        +string  description
        +datetime  date_created
        +List-string keywords
        +List-renku_api_Plan plans
        +List-renku_api_Mapping mappings
        +List-renku_api_Link links
        +bool  deleted
        +List-renku_api_Activity activities
    }
    class  Mapping {
        <<Connections hidden for readability>>
        + string name
        + string description
        + Path value
        + renku_api-Input_Output_Parameter_Mapping parameters
    }
    class  Link {
        <<Connections hidden for readability>>
        +renku_api-Output_Parameter source
        +List-renku_api-Input_Parameter sinks
    }
    class Activity {
        +datetime started_at
        +datetime ended_at
        +string-name-email user
        +dict annotations
        +List-renku_api_UsedInput used_inputs
        +List-renku_api_CreatedOutput created_outputs
        +List-renku_api_FieldValue parameters
        +renku_api_Plan base_plan
        +renku_api_Plan executed_plan
        +List-renku_api_Activity preceding_activities
        +List-renku_api_Activity following_activities
        +List-renku_api_Activity list()
        +List-renku_api_Activity filter_by_input()
        +List-renku_api_Activity filter_by_output()
        +List-renku_api_Activity filter_by_parameter()
    }
    class UsedInput {
      +string checksum
      +string path
    }
    class CreatedOutput {
      +string checksum
      +string path
    }
    class FieldValue {
      +renku_api-Input_Output_Parameter field
      +Any value
    }
    Dataset --> DatasetFile : files
    Project --> ProjectStatus : status
    ProjectStatus --> Activity : stale_activities
    Activity --> UsedInput : used_inputs
    Activity --> CreatedOutput : created_outputs
    Activity --> FieldValue : parameters
    UsedInput ..> Input
    CreatedOutput ..> Output
    UsedInput ..> FieldValue
    CreatedOutput ..> FieldValue
    Activity --> Plan : base_plan
    Activity --> Plan : executed_plan
    Activity --> Activity : preceding_activities
    Activity --> Activity : following_activities
    FieldValue ..> Parameter : field
    FieldValue ..> Input
    FieldValue ..> Output
    Plan --> Input : inputs
    Plan --> Output : outputs
    Plan --> Parameter : parameters
    Plan --> Activity : activities
    CompositePlan --> Plan : plans
    CompositePlan --> Mapping : mappings
    CompositePlan --> Link : links
    CompositePlan --> Activity : activities

    %% Mapping and Link connections have been hidden for imporved diagram readability
    %% Mapping --> Input : mapped_parameters
    %% Mapping --> Output : mapped_parameters
    %% Mapping --> Parameter : mapped_parameters
    %% Link --> Output : source
    %% Link --> FieldValue : source
    %% Link --> Input : sinks
    %% Link --> FieldValue : sinks

```

### Getting Activities and Plans

```Python
from renku.api import Activity

all_activities = Activity.list()
activities_by_input = Activity.filter_by_input(path="data/my.csv")
activities_by_multiple_input = Activity.filter_by_input(path=["a.csv", "b.csv"])
activities_by_filtered_input = Activity.filter_by_input(path=lambda p: p.endswith(".csv"))
activities_by_input_parent = Activity.filter_by_input(path="data/", exact=False)
activities_by_output = Activity.filter_by_output(path="data/my.csv", exact=True)
activities_by_multiple_output = Activity.filter_by_output(path=["a.csv", "b.csv"])
activities_by_filtered_output = Activity.filter_by_output(path=lambda p: p.endswith(".csv"))
activities_by_output_parent = Activity.filter_by_output(path="data/", exact=False)
activities_by_parameter = Activity.filter_py_parameter(name="lr", value=0.7)
activities_by_multiple_parameter = Activity.filter_py_parameter(name=["lr", "e"], value=[0.7, 1.0])
activities_by_filtered_parameter = Activity.filter_py_parameter(name=lambda n: n.startswith("lr."))
activities_by_filtered_parameter = Activity.filter_py_parameter(name="lr", value=lambda v: v > 0.5)
```

`.list()` returns all activities in the project.

`.filter_by_input()` and `.filter_by_output()` return all activities
that used a certain input or produced a certain output. Setting
`with_children=True` means that if a directory is passed, all activities for
that directory and its children are returned.

`.filter_py_parameter()` filters activities based on parameters' names and/or
values.

Parameters `path`, `name` and `value` should accept plain values, list of
values or a `Callable` accepting the value and returning a `bool`, to filter
on a single value, list of values or custom condition, respectively.

```Python
from renku.api import Plan

all_active_plans = Plan.list()
all_plans = Plan.list(include_deleted=True)
```

`.list()` returns all plans in the project. `include_deleted=True` would
also return plans that were deleted.

```Python
from renku.api import Project

status = Project().status()
status = Project().status(outputs=['my_output'])
```

`.status()` returns the current project status (`ProjectStatus`), detailing
detailing stale outputs and activities as well as modified and deleted inputs
that cause them to be stale. With `outputs` specified, limits status to those
outputs and activities producing them.

### Viewing the Inputs, Outputs, and Parameters of Plans and Activities
To see the names and default values of all the input, output, or parameters fields in a Plan:

```python
from renku.api import Plan

all_active_plans = Plan.list()
my_plan = all_active_plans[0]

inputs_list = my_plan.inputs
outputs_list = my_plan.outputs
parameters_list = my_plan.parameters

# print Input fields and their default values
for input_field in inputs_list:
    print(f"{input_field.name}: {input_field.value})
```



And to see the input or output filepaths associated with an activity:

```python
from renku.api import Activity

all_activities = Activity.list()
my_activity = all_activities[0]

input_files = my_activity.used_inputs
output_files = my_activity.created_outputs
for f in input_files:
    print(f"{f.path}")

parameters = my_Activity.parameters
for p in parameters:
    print(f"{p.field.name}: {p.value}")

```

To see the actual values that were specified for the input and output fields at the
execution of the activity (the input & output _field values_ may be different from
the input and output _paths_ if the fields were interpretted. For example, if an
`Output` was set to `my_file_{yyyy-mm-dd}.txt`, then the value in the
`CreatedOutput` list would be `my_file_2022-03-30.txt`):

```python
from renku.api import Activity

all_activities = Activity.list()
my_activity = all_activities[0]
executed_command = my_activity.executed_command
# "python myscript.py -p1=x --output=my-output"
```

### Classes

#### ProjectStatus

| API Property     | Wrapped property        | Type                     |
|------------------|-------------------------|--------------------------|
| stale_outputs    | from get_status_command | List[string]             |
| stale_activities | from get_status_command | List[renku.api.Activity] |
| modified_inputs  | from get_status_command | List[string]             |
| deleted_inputs   | from get_status_command | List[string]             |

#### Activity

| API Property | Wrapped Property | Type                           |
|--------------|------------------|--------------------------------|
| started_at   | started_at_time  | datetime                       |
| ended_at     | ended_at_time    | datetime                       |
| user         | agents (Person)  | string ('name (email)')        |
| annotations  | annotations      | dict                           |
| used_inputs  | usages           | List[renku.api.UsedInput]      |
| created_outputs      | generations      | List[renku.api.CreatedOutput] |
| parameters   | parameters       | List[renku.api.FieldValue] |
| base_plan    | association.plan | renku.api.Plan                 |
| executed_plan| plan_with_values.to_argv() | string (full command)|
| preceding_activities | lazy, through ActivityGateway | List[renku.api.Activity] |
| following_activities | lazy, through ActivityGateway | List[renku.api.Activity] |

##### UsedInput

| API Property | Wrapped property | Type   |
|--------------|------------------|--------|
| checksum     | entity.checksum  | string |
| path         | entity.path      | string |

##### CreatedOutput

| API Property | Wrapped property | Type   |
|--------------|------------------|--------|
| checksum     | entity.checksum  | string |
| path         | entity.path      | string |

##### FieldValue

| API Property | Wrapped property                                                           | Type                               |
|--------------|----------------------------------------------------------------------------|------------------------------------|
| field        | parent_activity.association.plan.(inputs,outputs,parameters)[parameter_id] | renku.api.(Input,Output,Parameter) |
| value        | value                                                                      | Any                                |

#### Plan

| API Property  | Wrapped property           | Type                      |
|---------------|----------------------------|---------------------------|
| name          | name                       | string                    |
| description   | description                | string                    |
| date_created  | date_created               | datetime                  |
| keywords      | keywords                   | List[string]              |
| inputs        | inputs                     | List[renku.api.Input]     |
| outputs       | outputs                    | List[renku.api.Output]    |
| parameters    | parameters                 | List[renku.api.Parameter] |
| command       | to_argv(with_streams=True) | string                    |
| keywords      | keywords                   | List[string]              |
| success_codes | success_codes              | List[int]                 |
| deleted       | invalidated_at is not None | bool                      |
| activities    | lazy, filter through ActivityGateway | List[renku.api.Activity] |

##### Input

| API Property  | Wrapped property              | Type                      |
|---------------|-------------------------------|---------------------------|
| name          | name                          | string                    |
| description   | description                   | string                    |
| value         | default_value or actual_value | string                    |
| prefix        | prefix                        | string                    |
| position      | position                      | int                       |
| mapped_stream | mapped_to.stream_type         | string                    |

##### Output

| API Property  | Wrapped property              | Type                      |
|---------------|-------------------------------|---------------------------|
| name          | name                          | string                    |
| description   | description                   | string                    |
| value         | default_value or actual_value | string                    |
| prefix        | prefix                        | string                    |
| position      | position                      | int                       |
| mapped_stream | mapped_to.stream_type         | string                    |

##### Parameter

| API Property  | Wrapped property              | Type                      |
|---------------|-------------------------------|---------------------------|
| name          | name                          | string                    |
| description   | description                   | string                    |
| value         | default_value or actual_value | Any                       |
| prefix        | prefix                        | string                    |
| position      | position                      | int                       |

#### CompositePlan

| API Property  | Wrapped property           | Type                      |
|---------------|----------------------------|---------------------------|
| name          | name                       | string                    |
| description   | description                | string                    |
| date_created  | date_created               | datetime                  |
| keywords      | keywords                   | List[string]              |
| plans         | plans                      | List[renku.api.Plan]      |
| mappings      | mappings                   | List[renku.api.Mapping]   |
| links         | links                      | List[renku.api.Link]      |
| deleted       | invalidated_at is not None | bool                      |
| activities    | lazy, filter through ActivityGateway | List[renku.api.Activity] |

##### Mapping

| API Property  | Wrapped property              | Type                                       |
|---------------|-------------------------------|--------------------------------------------|
| name          | name                          | string                                     |
| description   | description                   | string                                     |
| value         | default_value or actual_value | Path                                       |
| parameters    | mapped_parameters             | renku.api.(Input,Output,Parameter,Mapping) |

##### Link

| API Property  | Wrapped property              | Type                                       |
|---------------|-------------------------------|--------------------------------------------|
| source        | source                        | renku.api.(Output,Parameter)               |
| sinks         | sinks                         | List[renku.api.(Input,Parameter)]          |

## Drawbacks

While we can use indices for `filter_by_usage` and `filter_by_generation`,
plain `Activity.list()` on a large repository could potentially use many
resources and not be very efficient.

Most filtering with this solution has to be done by the user in their code, as
we don't provide a generic query interface that could benefit from indices.

## Rationale and Alternatives

While it is a bit on the simple side, it should cover all use-cases and map
very closely to our internal implementation, making it easy to maintain and
extend.

I also considered adding something with a custom query DSL, e.g.

```
Activity.list({"usages": {"path": "data/my.csv"}})
```

or

```
Activity.list(where="usages.path in 'data/my.csv' and started_at > '2022-01-01'")
```

but that would add quite a bit of overhead in development time and force users
to learn the DSL, adding cognitive overhead.

Not implementing this would mean users have to shell out to renku and parse
data from the CLI or call `renku.command` methods or gateways directly, which
may change their interface and aren't easy to understand.


It was also considered to return a plan with values applied from an activity,
essentially `Activity.plan_with_values` from renku.core, but this was dropped
in favor of just returning the argv string of the activity, to not
overcomplicate things.

## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

We should settle on the names of properties, so they're as descriptive as possible.


> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

This RFC is only concerned with offering information about Activities and Plans
to the user, so it only considers reads. At some point we might also consider
supporting writes, namely creating new (Composite)Plans in scripts and possibly
executing plans through scripts. In addition, it might also be nice if we could
generate YAML files for execution based on values a user sets on a
(Composite)Plan instance.
