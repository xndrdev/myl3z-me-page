---
title: Open WebUI
---

# Open WebUI

I want to set up [Open WebUI](https://openwebui.com/) at home and experiment with local
AI models. The first step is finding out how well it runs on my hardware and where it can
help in everyday life. One concrete goal is already clear: make my documentation available
through a chat that answers questions and points to its sources.

## The idea

As my [homelab]({{< relref "/posts/homelab" >}}) grows, so does the amount of information
to keep track of. Which services run on which machine? Which network does a device belong
to? Where is the configuration, and why did I set it up that way?

My [Linux &amp; Server]({{< relref "/docs/linux" >}}) and
[network]({{< relref "/docs/network" >}}) notes are a starting point, with more documentation
from home to follow. Questions such as “Where does my DNS server run?” or “Where did I
document the VLAN layout?” should lead to the relevant information, including the document
and passage so I can check the answer.

## Documentation as a knowledge base

Open WebUI can collect documents in a
[Knowledge Base](https://docs.openwebui.com/features/workspace/knowledge/) and make them
available to chats. I want to try RAG, *Retrieval-Augmented Generation*, for searching them.
It finds relevant passages in the documents and gives them to the model as context for
its answer. Open WebUI supports
[source citations in responses](https://docs.openwebui.com/features/chat-conversations/rag/).

To answer “What is where right now?”, the knowledge base needs to stay current. Changes to
services, devices, and configurations need to go into the documentation first and then be
carried over to Open WebUI. Working out that update process is part of the project. The
chat alone cannot detect changes to my home network.

## Planned setup

| Component | Purpose |
| --- | --- |
| Open WebUI | Chat interface and knowledge base management. |
| Ollama with a local language model | Generate answers on my own hardware; I will experiment to find a suitable model. |
| Local embedding model and RAG | Prepare documents for semantic search and find relevant passages. |
| Docker Compose | Run the local setup with persistent data. |

The [official Quick Start](https://docs.openwebui.com/getting-started/quick-start/) covers
running with Docker Compose and connecting to Ollama. In my setup, both answer generation
and document processing should run locally.

## The first experiment

I will start with a few well-maintained documents and questions whose answers I already
know. That lets me check whether it finds the right passage, whether the source supports
the answer, and how the model handles missing or contradictory information. My goal is
for it to acknowledge gaps in its knowledge.

The project is still in planning. This is where I will document the setup, experiments,
and uses that emerge along the way.
