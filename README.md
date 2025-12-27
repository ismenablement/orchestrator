# Orchestrator

A simple orchestrator application.
Idea of this small project is to test ability to run Orchestrator flow, and trigger using workflow_dispatch worklows in the repoA/B/C. 
In real life, repoA/B/C hold Java code and need to build application, but running build processes on each repo needs to be done in spepcific order. 
Idea is to trigger repoA/B/C build workflow_dispatch, and then monitor status by pulling from API. It will be TriggerAndWait kind of reusable workflow that we will create for this purpose. 
Main Orchestrator workflow, living in Orchestrator repo, will have simple and straight logic that will allow running repoA/B/C build in sequential and/or parallel order by defining needs:. 

Note: Alarady analyzed option to do event-based driven pipeline (using workflow_run to detect when child job is finished, and sending repository_dispatch event back to the Orchestrator), but figured out that although it makes sense from cost perspective it is complex logic and confusing GH Acitons runs. 

