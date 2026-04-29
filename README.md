# Ensemble Session Inspection Agent

> Hackathon 2026 — planned but not selected for implementation. To be built in a future sprint.

## What Is This?

The **Ensemble Session Inspection Agent** is an AI-powered diagnostic tool for InterSystems Ensemble/IRIS Interoperability productions. Given a session ID, it correlates message headers, message bodies, the event log, rule log, and Business Process source code to produce a plain-English narrative of what happened — replacing 20-30 minutes of expert tab-switching with a 30-second conversation.

This project was designed and planned during the READY 2026 internal hackathon. The team chose a different project for the hackathon implementation sprint, but the Ensemble Session Inspection Agent is intended for future development.

See [`_bmad-output/planning-artifacts/`](./_bmad-output/planning-artifacts/) for the full product brief, PRD, architecture doc, and epics that were produced during planning.

---

# READY Hackathon Dev Template

This repo is based on the hackathon dev template, which provides a template to kickstart development with AI Hub.

## Contents

- **./skills** - agent skills with information on using AI hub for AI agents. Move these to a suitable location for your preferred AI coding agent. 
- **./src/Sample** - Basic sample classes for tools, toolsets, agents and MCP servers. These are installed with zpm when the container is build.  
- **./src/Python** - An example stdio MCP server defined in Python and used in the IRIS Toolsets 
- **Datasets.md** - Notes on some datasets or tools to import datasets available on Open Exchange for easy install 

## Quickstart

### Download AI Hub Container
Download an AI Hub container from the [Early Access Program Portal](https://evaluation.intersystems.com/Eval/early-access/AIHub). The docker-containers end with `docker.tar.gz`, ensure you choose the version suitable for your operating system (arm64 for macOS).

Then load the image with: 

```bash
docker load -i /path/to/iris-community-2026.2.0AI.147.0-docker.tar.gz
```

After this, you can run:

```bash
docker images
```

And you should see an image called `docker.iscinternal.com/docker-intersystems/intersystems/iris-community:2026.2.0AI.147.0`. If you have downloaded an arm64 container, this tag will be slightly different. **Make sure to change the tag at the top of the Dockerfile to the correct one for your operating system**.

If you have a license key and want to use a licensed version, again ensure to download the correct container for your operating system, then change the tag at the top of a docker file. In this case, you will also need to run web-gateway in the docker-compose project. 

### Build Template Repo

After this you can clone this repo: 

```bash
git clone https://github.com/intersystems-community/ai-hub-dev-template
cd ai-hub-dev-template
```

First, in order to use the Agent demo, add an OPENAI_API_KEY to a file called .env in this repo. You can see an example in .env.example.

Then build the project with: 

```bash
docker-compose up -d --build 
```

## Accessing IRIS 

You can find the Management Portal at http://localhost:52773/csp/sys/UtilHome.csp.

Login with: 
    - SuperUser / SYS

You can access the IRIS Terminal with:

```objectscript
docker-compose exec -it iris iris session iris
```

or the bash terminal with:

```bash
docker-compose exec -it iris iris session iris
```


## Testing Sample agent 

There is a basic agent in src/Sample.Agent, a simple way to use it from objectscript is to run the following (note this does require an OPENAI_API_KEY to be added to .env before running th container). 

```objectscript
set $NAMESPACE= "IRISAPP"
Set agent = ##class(Sample.Agent).%New()
Set sc = agent.%Init()
write:sc'=1 $SYSTEM.Status.GetErrorText(sc), !

Set session = agent.CreateSession()

// This requires using both tools defined in Sample.Tools and packaged in Sample.ToolSet
Set request = "Add a person named Alice aged 30, and then get people younger than 35."
Set response = agent.Chat(session, request)
write response.content
```

## Test MCP Server

The build process installs an MCP server web application at http://localhost:52773/mcp/sample. You can check this MCP server is running by going to http://localhost:52773/mcp/sample/v1/services. 

For the MCP Server to be usable, there is an additional step of starting this via a Rust binary which connects to IRIS through the web gateway protocol. The Binary is installed in `/usr/irissys/bin` (should already be in PATH).  

A sample configuration is shown in [config.toml](./config.toml), which serves a remote HTTP server on port 8080 (which is exposed by the docker-compose file). 

To start the transport, open a bash terminal within the container: 

```bash
docker-compose exec -it iris bash 
```

Then start the `iris-mcp-server`

```bash 
iris-mcp-server -c config.toml run 
```

You can now connect the MCP server to your MCP Client of choice (e.g. coding agents like claude code) using the address: http://localhost:8080/mcp/sample. 

An example python MCP client is shown in test_mcp_connection.py, which uses Langchain's MCP adapters module. To try this, run: 

```bash
pip install langchain-mcp-adapters
python test_mcp.py
```