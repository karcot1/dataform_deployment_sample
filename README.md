# dataform_deployment_sample
Sample code to demonstrate how Dataform pipelines can be set up and scaled using Cloud Build and Pub/Sub.

## About this project

**Overview**

This code is intended to serve as an example of how to set up Dataform deployments that can easily scale to become complex but manageable pipelines. When executed, Dataform will create three separate subject areas in sequence: account, customer, and sales.

By creating a Cloud Build trigger for each build, we ensure that any one subject area can also be built on its own in addition to being part of a larger pipeline.

Each build consists of a series of "dataform run" executions that create and run BigQuery jobs. The orchestration of these jobs is all handled by Dataform, based on the SQLX code stored in this repository. The final step in each build pushes a "success" message to the Pub/Sub topic, with metadata specifying which subject area has just completed.  

The pipeline executes in the following order: account --> customer --> sales

The final step in "account" is to publish a message to the Pub/Sub topic "dataform-deployments". The customer build is a Pub/Sub invoked trigger that is subscribed to the dataform-deployments topic. When the message from account comes through, it fires off the build for "customer". Similarly, the final step of "customer" is to publish a message to the same topic, which kicks off the "sales" build.

Note: the "account" build contains one extra step where it creates the sample source tables used in this example.

**Tags**

In this example project we split up the SQLX code using Tags. In the cloudbuild.yaml files, we then specify the Dataform execution specifically by calling out the appropriate tag for each build. This ensures that none of our SQLX files overlap or get run when they are not supposed to. As a result, it is recommended to use tags when building out a warehouse with multiple distinct workflows.

**Services Used**

This code relies on the following GCP services:
- Cloud Build
- Pub/Sub
- BigQuery

This code also makes use of the Dataform CLI.

**Prerequisistes**

Before running this code, please ensure you have completed the following steps:

1. Edit dataform.json to ensure "defaultDatabase" is pointed at your GCP project
2. In this GCP project, create a service account with the following permissions:
    - BigQuery Admin
    - Logs Writer
3. Ensure that Cloud Build, Pub/Sub, and BigQuery APIs are enabled
5. In Pub/Sub, create a topic called "dataform-deployments"
4. Connect this repository (or a forked version of it) in Cloud Build. Region does not matter, but the build triggers must be created in the same location as your repository.

**How to Use**

In your GCP project, create a cloud build trigger for each sample subject area. Create a manual trigger for "account", and a Pub/Sub trigger for "customer" and "sales".

Set each trigger to connect to this repository, and point it towards the appropriate YAML configuration file (e.g., for "account", make sure the trigger points to "cloud_build_account.yaml").

For the Pub/Sub invoked triggers, subscribe to the "dataform-deployments" topic you created in the previous section. 

For all three triggers, set the following subsitution variables:
- _PROJECT_ID --> your GCP project
- _BQ_LOCATION --> US

For the two Pub/Sub invoked triggers (customer and sales), you will need to set a third substitution variable:
- _SUBJECT_AREA --> $(body.message.attributes.subjectArea)

This is where the tight integration between GCP services comes in handy - Cloud Build can access Pub/Sub metadata (such as attributes) of messages in the topic it listens to. If you review the final steps of all three YAML files, you can see that we pass the subject area being built as an attribute.

Next, we can add a filter to the pub/sub triggers based on the values in this metadata.

Set the filter to be for _SUBJECT_AREA, and set the value to be whichever subject area is directly *upstream* of the current trigger. 

In this example, we want "customer" to run only after "account" has completed. As a result, the filter in the customer build trigger should be waiting for "ACCOUNT". Subsequently, the sales build trigger should be waiting for "CUSTOMER".

This is how we can link the triggers to fire at just the appropriate time, instead of every single time a message is published to the specified topic.

**Running the Code**

Once all three triggers have been created, you can manually trigger "account" or set it to execute on a schedule. Monitor the run and verify that once account completes, "customer" automatically kicks off. Once "customer" completes, you should see "sales" kick off next.

## Next Steps

**Scaling existing subject areas**

Existing subject areas can be scaled with no impact to the deployment or orchestration we have set up. The SQLX is version controlled in GitHub, so multiple engineers and analysts can work on the same subject area together by creating feature branches. Once a feature is ready to be merged, a PR into master will integrate the changes, and Cloud Build will automatically be able to pick up and execute the updated Dataform code.

NOTE: If you would like to expand any subject area into more steps or create a more complex workflow, just ensure that all the SQLX files for that subject area have the appropriate tag in their config block.

**Adding new subject areas**

Adding new steps to your DAG is also a straightforward process - the workflow itself can be created in SQLX files using Dataform. Like the existing build, ensure that all files in this new workflow have a tag in their config block that ties them together. 

Once the workflow is complete and has been tested locally, it can be integrated into your overall build by adding a new YAML file that follows the same structure as the existing ones, and creating a new Cloud Build trigger.


Just be sure to change the DATAFORM_TAGS value to point to your new workflow, and update the subjectArea attribute of the final Pub/Sub step.

**CICD with Dataform**

This workflow can also be expanded into a more robust CICD implementation - consider the following example as a starting point:

- Create three GCP projects for each environment (dev, stage, prod) and three branches in GitHub to reflect this. Update dataform.json to point to the appropriate project in each branch.

- Create the same triggers in each project, but specify the appropriate branch to use.

- Have developers create feature branches off of prod and create changes. Merge the changes into dev to test, then into stage, then into production.



