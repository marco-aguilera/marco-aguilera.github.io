---
title: "We Were Context Engineering Before We Called It Context Engineering"
date: 2026-11-09
draft: false
summary: "What reconfiguring a Kubernetes Operations Agent taught our team about the importance of context and getting reliable answers."
tags:
  - Elastic
  - Context Engineering
  - Agent
  - Tools
  - Kubernetes
---


This year, I had the opportunity to attend ElasticON, and one of the topics that sparked my interest was Context Engineering for AI Agents. The discussion focused on an issue that was very familiar to what we were experiencing at an enterprise level. We know that AI Agents can spend a lot of time processing information, understanding the schema, and deciding how to query data, which can drive up the token bill significantly.
I found this topic interesting because we recently encountered a similar issue when building a Kubernetes Agent on Elastic.
During the event, Elastic presented its Context Engine as a layer that would contain the data ahead of the prompt. This solution created a reusable unit of knowledge called a Knowledge Indicator and pulled the relevant information for an agent instead of having the agent repeatedly search and query for the same information. Elastic published an example that reduced calls from 33 to 9 and input tokens by 71% while maintaining the correct full answer.
Our implementation was not a complete Context Engine, but I noticed that many of the engineering decisions that were made in Agent Builder were solving the same issue.


**Removal of Agent Rediscovery:**

Before: User Question → Agent determines data source → Agent discovers fields → Agent creates query → Agent interprets results → Answer

After:
User Question → Agent determines intent → Selects tested tool → Tool retrieves known data correctly → Agent reasons over results → Answer

Elastic provides a way to build your own tools with ESQL, which will query for specific data from your enterprise environment and capture the correct data source, fields, query, calculations, and units. This returns more reliable answers.
Example of one of the Elastic tools:

**Kubernetes-Restarts**

```sql
FROM metrics* METADATA _index | WHERE @timestamp >= NOW() - 24 hours AND STARTS_WITH(_index, CONCAT(".ds-metrics-", ?cluster, "-")) AND kubernetes.namespace == ?namespace AND kubernetes.container.name IS NOT NULL | STATS max_restarts = MAX(kubernetes.container.status.restarts), last_reason = MAX(kubernetes.container.status.last_terminated_reason) BY kubernetes.pod.name, kubernetes.container.name | SORT max_restarts DESC NULLS LAST | LIMIT 25
```

The tool above encapsulates the stats of the restarts and last terminated reason by the pod name and container name and proceeds to sort them with a limit of 25.
This is an example of how this tool not only supports the agent in its findings by gathering precise information but also helps the agent leverage this built tool to call for that information to attain that context before the LLM starts to analyze anything else.



**Context Is More Than Data:**

I realized that several kinds of context can complement the LLM before it performs searches and reasoning to ultimately produce a more complete answer and reduce the number of tokens it uses to come to a conclusion.
Contextual information like "Where does the Kubernetes CPU data live?" or "Which fields identify the namespace?" are some examples, but there are many more. In our implementation, we also ensured there was semantic contextual information about what certain values meant, helping the LLM know that "Missing restart telemetry" did not become "0 restarts."
The other part of the contextual support we gave the agent was providing the rules on how to investigate the information by isolating different tools that point toward logs or metrics. We have the agent use those tools to run checks on resources, restarts, or errors before analyzing the information and recommending the next investigation.


**The Results:**

Before the rebuild, the agent failed the validation scenarios. The agent would write its own queries each time, which might not be entirely correct, and could spend too many tokens. It was not able to see all the available clusters. There was a broken pattern-matching function that would cause stderr noise to be counted as errors. There was a consistent report of 0 even when there was no data or the results were empty. There was also a security concern because it could echo secrets from logs that needed to be redacted.
Once the rebuild was completed with the support of Elastic, the Agent began to perform reliably and allowed the agent to find the correct data using the built-in tools that were designed to be read-only, with secrets and client IDs masked as redacted to ensure better security. The Agent now calls 15 fixed and tested tools and can additionally make all the clusters visible to the user, with scope into real errors. The agent's tools give it the ability to pull reliable information with restart reasoning.


- k8s_list_clusters → Which clusters exist? → All AKS clusters sending logs, with volume
- k8s_find_namespace → Which cluster is namespace X in? → Matching clusters; asks you to pick if several
- k8s_namespace_ranking → Which namespace uses the most CPU / memory? → Cores, GB and pods per namespace
- k8s_log_volume → Which namespace is busiest? → Log lines and lines/min per namespace
- k8s_services → How many services? Pods per service? Limits? → Pods, CPU/memory min/avg/max and derived limits per service
- k8s_pod_resources → How are the pods doing? → CPU, memory, % of limit, service and node per pod
- k8s_node_resources → How are the nodes doing? → CPU, memory and pod count per node
- k8s_restarts → Any restarts or crash loops? → Restarts since pod start and last exit reason
- k8s_errors → What errors are there? → Error lines, HTTP 5xx and a masked sample per container
- k8s_error_patterns → Why is it failing? → Most frequent error messages and pods affected
- k8s_pod_logs → Show me the logs of pod X → Latest 50 lines, secrets masked
- k8s_cpu_timeseries → Chart CPU / memory for a namespace → % of limit over time
- k8s_service_cpu_timeseries → Chart CPU per service → Millicores per service over time
- k8s_service_mem_timeseries → Chart memory per service → MB per service over time


This issue was not solved by replacing the LLM with a more powerful model. The issue was solved by improving the system around the model by giving the Agent more contextual information about the data it was going to investigate. We also solved this issue by giving the Agent tools that reliably and correctly produce factual conclusions, which ultimately reduces token usage by letting it reason with the context it gathers.


**How ElasticON Changed How I Thought About It:**

The discussion at ElasticON regarding context engineering provided an understanding of some of the information we provided to the specific agent and how we could improve this once our team had the clusters on a higher version and could leverage the Context Engine. This would allow us and other teams to have their source data help their specific agents gather context about the data and also reduce token spending since the LLM does not have to search and investigate for the same information repeatedly.

**Governance:**

Using these context strategies to reduce token count also connects to governance around the agents that are created for different teams. Our goals of reducing token count and spending can also be achieved by setting restrictions on the agents. Who has access to the agent? How many tokens are they allowed to spend? What types of questions are users allowed to ask, and what questions are allowed to be answered? These are security filters that can be placed on the agents to provide granular control over the agents and token costs.
One of the concerns that we have run into was setting up these controls so that only dedicated users have access to the specific agents needed for their use cases. These are also strategies that can help reduce your enterprise token usage and ensure tokens are only used for necessary work.

**Lessons Learned:**

- Don't make the LLM discover what the platform already knows.
- Providing relevant and reliable context is better than a large volume of context.
- Tools and agent probabilistic reasoning can complement each other in investigations.

**Observations:**

When Elastic supported us in the Kubernetes Operations Agent rebuild, we noticed the improvements came from configuring the environment around the model and not changing the model itself.

We are also reducing what the agent is discovering by giving it reliable tools and, within the instructions, establishing clear rules.

Initially, I thought about this as making the agent more reliable, but after hearing the context engineering discussion at ElasticON, I see its true benefit.

