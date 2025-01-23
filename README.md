# Resources for management of a Knime Analytics Platform headless execution on a Windows Server

The idea here is to have a Knime Analytics Platform on a Windows Server performing some management activities, like:

1) Knime Analytics Platform version management
2) headless execution of Knime workflows
3) execution logs management
4) workflow scheduling

Yes, you can do these things and much more using Knime Community/Business Hub. But what about you can't pay for these tools ? For instance, here in Brazil, where 38% of work force [earn ~US$ 291/month](https://countryeconomy.com/national-minimum-wage/brazil), the [~US$ 40k of a yearly license of a Basic Knime Business Hub](https://www.knime.com/knime-hub-pricing) are prohibitive !

## AS IS scenario

How Knime Server works:

- The analyst develops the workflow in the Knime Analytics Platform on his/her local workstation
- The workflow is published and tested in the Knime Server Homologation environment (a homologation folder on Knime Server with specific permissions)
- The tested workflow is published in the Knime Server Production environment (a production folder on Knime Server with specific permissions)
- The analyst schedules and monitors the execution of the workflow while connected to the Knime Server
- Data apps are accessed by the end user in the Knime Server web interface

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/knime_server_AS_IS_EN.png">
 <source media="(prefers-color-scheme: light)" srcset="images/knime_server_AS_IS_EN.png">
 <img alt="AS IS scenario" src="images/knime_server_AS_IS_EN.png">
</picture>

## TO BE scenario

How the Knime Server on Windows works:

- The analyst develops the workflow in the Knime Analytics Platform on his/her local workstation
- The workflow is versioned for a git repository (Github, Gitlab)
- The versioned workflow is imported into Knime on the Windows server, in the environment (homologation or production) and in the folder indicated by the Analyst
- The workflow execution is scheduled following the configuration proposed by the Analyst
- The execution logs of the production environment are published in a web interface for consultation by the Analyst

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/knime_server_TO_BE_EN.png">
 <source media="(prefers-color-scheme: light)" srcset="images/knime_server_TO_BE_EN.png">
 <img alt="TO BE scenario" src="images/knime_server_TO_BE_EN.png">
</picture>

## Not supported

The following features are not supported in Knime on Windows:

- Execution of Data Apps
- Saving workflows with errors

What is currently not supported by Knime on Windows Server but could be implemented:

- Segregation of logs by environment (homologation and production)
- Notification of execution with errors by email

Now let's dive into the implementation details.

## Detailed processes

Below are the processes implemented in Knime on Windows Server:

[1 – Analyst produces the WF and publishes it on git repo](#1_-_analyst_produces_the_wf_and_publishes_it_on_git_repo)

[2 – WFs from the main branch are available](#2_–_wfs_from_the_main_branch_are_available)

[3 – Import of WFs from the main branch and make them available in the local Knime workspace](#3_–_import_of_wfs_from_the_main_branch_and_make_them_available_in_the_local_knime_workspace)

[4 – Knime Analytics Platform on Windows Server accesses the WFs](#4_–_knime_analytics_platform_on_windows_server_accesses_the_wfs)

[5 and 6 – Scheduled WFs are executed independently](#5_and_6_–_scheduled_wfs_are_executed_independently)

[7 – WF execution logs are stored](#7_–_wf_execution_logs_are_stored)

[8 – Stored logs are converted to HTML](#8_–_stored_logs_are_converted_to_html)

[9 – Web server makes HTML log pages available](#9_–_web_server_makes_html_log_pages_available)

[10 - Time based orchestrator](#10_-_time_based_orchestrator)


<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/knime_on_windows_how_things_work_EN.png">
 <source media="(prefers-color-scheme: light)" srcset="images/knime_on_windows_how_things_work_EN.png">
 <img alt="Technical details about Knime on Windows" src="images/knime_on_windows_how_things_work_EN.png">
</picture>

## 1 – Analyst produces the WF and publishes it on git repo

In order for the workflow to be considered in the Knime execution process on Windows, some precautions must be followed by the developer:

- [Develop the workflow within the caller-caller model](#develop_the_workflow_within_the_caller-caller_model)
- [Provide a scheduling and notification file](#provide_a_scheduling_and_notification_file)
- [Publish the workflow and the notification scheduling file](#publish_the_workflow_and_the_notification_scheduling_file)

### Develop the workflow within the caller-caller model

The caller-caller execution model is adopted as part of the data science application lifecycle, presented by the company Knime as follows:

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/knime_life_cicle.png">
 <source media="(prefers-color-scheme: light)" srcset="images/knime_life_cicle.png">
 <img alt="Data science creation and productionization model. Source: https://www.knime.com/blog/how-to-move-data-science-into-production" src="images/knime_life_cicle.png">
</picture>

The Knime Analytics Platform has a set of resources (nodes) classified as Workflow Services, aimed at deploying applications. The caller-caller model separates the content, the logic, from the executor:

**CALLE** - workflow specialized in implementing the business logic, that which needs to be executed
**CALLER** - workflow specialized in invoking other workflows, handling errors and documenting them

This model is also useful for development testing and for deploying applications in Knime Hub. See ["KNIME Workflow Invocation Guide"](https://docs.knime.com/latest/analytics_platform_workflow_invocation_guide/analytics_platform_workflow_invocation_guide.pdf) for more details.

To develop a workflow (call) that will later be invoked by the caller, the Analyst may adopt the following development template:

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/calle_template.png">
 <source media="(prefers-color-scheme: light)" srcset="images/calle_template.png">
 <img alt="Calle template" src="images/calle_template.png">
</picture>

The nodes that implement the business logic have to be positioned between and connected to the "Workflow Service Input" and "Workflow Service Output" nodes, as in the following example:

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/calle_example.png">
 <source media="(prefers-color-scheme: light)" srcset="images/calle_example.png">
 <img alt="Calle example" src="images/calle_example.png">
</picture>

### Provide a scheduling and notification file

After developing the calle workflow, the Analyst must create a text file in JSON format containing the execution instructions.

This file must have the same name as the .knar or .knwf file to be generated, followed by the .json extension, and its content must follow the following format:

```
{
  "workflow_name": "CAPELANIA/LISTAGEM_ANIVERSARIANTES_DO_MES_EXCEL",
  "workflow_path": "/",
  "workflow_schedule": {
      "daily":  [ 
                    {"hour": "12", "notify": "user1@domain.com;user2@domain.com"},
                    {"hour": "15", "notify": "user1@domain.com;user2@domain.com"}
                ],
      "weekly": [	{"day": "Wednesday",
                    "schedule": [
                                    { "hour": "07","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "11","notify": "user1@domain.com;user2@domain.com"}
                                ]
                    },
                    {"day": "Friday",
                    "schedule": [
                                    { "hour": "08","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "12","notify": "user1@domain.com;user2@domain.com"}
                                ]
                    }
                ],
      "monthly": [	{"day": "16",
                    "schedule": [
                                    { "hour": "06","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "13","notify": "user1@domain.com;user2@domain.com"}
                            ]
                    },
                    {"day": "22",
                    "schedule":  [
                                    { "hour": "09","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "16","notify": "user1@domain.com;user2@domain.com"}
                            ]
                    }
                ]
  }
} 
```

For the example above, the file containing the workflows is called CAPELANIA.knar, and the scheduling and notification file is called CAPELANIA.json .

The content of the .json file must be valid JSON and contain the following objects:

**1) workflow_name** - string containing the full path to the only workflow that will be executed at the scheduled time. It is assumed that the .knar file will contain the same structure informed in this workflow_name attribute. If more than one workflow must be executed, create a main workflow and inform the path to this main workflow. For example: the CAPELANA.knar file was generated containing the main folder (CAPELANIA) and within it two objects (data folder and workflow LISTAGEM_ANIVERSARIANTES_DO_MES_EXCEL):

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/export_workflow.png">
 <source media="(prefers-color-scheme: light)" srcset="images/export_workflow.png">
 <img alt="Workflow name" src="images/export_workflow.png">
</picture>

And when the file CAPELANIA.knar is imported into Knime, its structure will be [DESTINATION DIRECTORY]/CAPELANIA:

<picture>
 <source media="(prefers-color-scheme: dark)" srcset="images/destination_folder.png">
 <source media="(prefers-color-scheme: light)" srcset="images/destination_folder.png">
 <img alt="Destination" src="images/destination_folder.png">
</picture>

The **workflow_name** object must contain the following value:

"workflow_name": "`CAPELANIA/LISTAGEM_ANIVERSARIANTES_DO_MES_EXCEL`"

**2) workflow_path** - for future use

**3) workflow_schedule** - an array of schedule objects: an array containing daily schedules, an array containing weekly schedules, and an array containing monthly schedules. 

For example, consider the array of the workflow_schedule object:

```
"workflow_schedule": {
      "daily":  [ 
                    {"hour": "12", "notify": "user1@domain.com;user2@domain.com"},
                    {"hour": "15", "notify": "user1@domain.com;user2@domain.com"}
                ],
      "weekly": [	{"day": "Wednesday",
                    "schedule": [
                                    { "hour": "07","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "11","notify": "user1@domain.com;user2@domain.com"}
                                ]
                    },
                    {"day": "Friday",
                    "schedule": [
                                    { "hour": "08","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "12","notify": "user1@domain.com;user2@domain.com"}
                                ]
                    }
                ],
      "monthly": [	{"day": "16",
                    "schedule": [
                                    { "hour": "06","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "13","notify": "user1@domain.com;user2@domain.com"}
                            ]
                    },
                    {"day": "22",
                    "schedule":  [
                                    { "hour": "09","notify": "user1@domain.com;user2@domain.com"},
                                    { "hour": "16","notify": "user1@domain.com;user2@domain.com"}
                            ]
                    }
                ]
  }
```

In this example, the workflow will be executed:
- **daily**, at 12:00 and 15:00, and also
- **weekly**, on Wednesdays at 07:00 and 11:00 and on Fridays at 08:00 and 12:00, and also
- **monthly**, on the 16th, at 06:00 and 13:00 and on the 22nd, at 09:00 and 16:00

**IMPORTANT:**
- The times must be in the range 00 to 23, represented with two digits
- The days of the week must be written in English, with the first letter capitalized and the rest lowercase
- The days of the month must be in the range 01 to 31, represented with two digits

The **notify** property must contain the emails (one or more, separated by semicolons) of the people who should be notified when errors occur in the execution of the workflow. This functionality has not yet been implemented.

### Publish the workflow and the notification scheduling file

After developing the workflow and creating the scheduling and notification file, the Analyst must:

1) Export the workflow or the folder containing the workflows and subfolders

The content of what was developed in the Knime Analytics Platform must be exported, generating either a .knar or a .knwf file.

If the project contains subdirectories/subfolders and/or other workflows, the export must contain these subdirectories and/or workflows:


## 2 – WFs from the main branch are available
## 3 – Import of WFs from the main branch and make them available in the local Knime workspace
## 4 – Knime Analytics Platform on Windows Server accesses the WFs
## 5 and 6 – Scheduled WFs are executed independently
## 7 – WF execution logs are stored
## 8 – Stored logs are converted to HTML
## 9 – Web server makes HTML log pages available
## 10 - Time based orchestrator

## Implementation information

Knime Analytics Platform version: 5.4


## Final remarks