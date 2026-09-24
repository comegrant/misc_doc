# DS Setup*
We're looking to revamp our Data Science setup to improve the developper experience and improve our capabilities.
We've decided to run a PoC on both Azure and Databricks to decide which direction to take.

Note that the best setup is likely to be a combinations of the two. With logic living in Databricks and Azure components being used for an integration layer (for service bus etc.)

## Current Pain Points
- Remote compute takes a long time to start, making iteration slow and tedious
- Lack of parity between local and remote environment causes bugs & amplify pain point above
- Internal packages are not versioned, making maintenance difficult. -> This is an objective in itself and doesn't need to be included in the PoC.

## Functional Requirements
The PoC should cover:
- Running python batch workloads on a schedule
- Reading & writing data to and from Databricks or the underlying storage layer.
? Running long-lived apps (Streamlit, FastAPI...) + with stable URL
? Event-driven workloads
? Real-time inference

## Non-functional Requirements

### Developper Experience
- Startup time / time to user code: Benchmark Azure & Databricks on first-time compute provisioning as well as hot, warm & cold starts at 8GB (ACA consumption profile), 16GB (Databricks Serverless Compute) and >16GB memory (Databricks Classic Compute or ACA Dedicated Profile)
- Environment Parity: It should be easy to replicate the remote environment locally, Data Scientists should feel confident that code running locally will work in production.
- Dependency Management: Adding libraries to projects should be easy and conflict-free.
- Deployment: Simple CLI commands should let Data Scientists start new projects and deploy them to dev / prod.
- CI/CD: Easily test and deploy changes with Github actions.
- Maintenance: Infrastructure changes can be done in-place with minimal downtime and manual work

### Performance
? Latency requirements for event-driven workloads?
? Latency requirements for real-time inference?

### Cost
? Where does cost stand in our list of priorities?
? How much time do we want to spend on getting accurate cost predictions?

### Security
- Access to deployed apps & jobs should be restricted to Cheffelo employees (most likely this means VPN/office wi-fi access only)
- Data Science projects should have user-assigned managed identities by default. Minimize secrets.
? It should be easy to control access to deployed apps (Is that true or are all ML apps accessible to all employees by default?)
? Keeping dependencies up-to-date and patching vulnerabilities should be easy and automatable.
