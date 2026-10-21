## Progent: Securing AI Agents with Privilege Control





### Abstract

**Background: **

AI agents interact with external environments through tool calls, exposing them to attacks like indirect prompt injection that can trigger unauthorized actions.



**Challenges:**

Securing these agents is challenging: 

- they behave autonomously and probabilistically, 
- security requirements evolve depending on the user’s task and execution state, and
- there is an inherent tradeoff between security and utility.

> Agents 通过工具调用可以与外部环境交互，但是这种工具调用的方式会把自己暴露在一些攻击当中，例如间接提示词注入，而这会触发非权限行为。
>
> 但是针对Agent的保护面临着一些挑战：
>
> - 这些工具调用的行为是自发的、有概率的
> - 安全需求会随着用户任务和执行状态变化
> - 这是一个内在的 安全性 与 实用性的权衡

Progent represents privilege as a security policy consisting of symbolic rules over tool names and arguments.

These rules specify which tool calls are allowed for task completion and which unnecessary ones are blocked for security.

![001](./note_1.assets/001.png)

> Progent 把 权限 作为 一个安全策略，而这个安全策略 由一些 符号规则组成，这些符号规则在 工具名称和参数之上。这些规则会明确区分 哪些工具调用是任务完成所应允许的 哪些工具调用是非必要的，并且为了保证安全应当阻塞的。

Every tool call is checked against such a policy through a deterministic procedure, enforcing the principle of least privilege.

> 每一个工具调用都会与 安全策略进行对照，这个过程用一个 确定的程序完成，使得每一个工具调用都执行这个最低权限原则。
>
> 我个人认为，这个“每一个工具调用都会与 安全策略进行对照” 设计，就是对第一个问题  “这些工具调用的行为是自发的、有概率的”的一种解决，也就是 不论工具调用何时出现，只要出现 就必须 与安全策略进行对照，一刀切 去 对抗自发性、随机性。



To handle diverse user tasks and evolving execution contexts, an LLM automatically generates the initial policy from the user’s task and updates it during execution as new information arrives.

>为了处理多个用户任务 和 不断变化的执行上下文，一个LLM根据用户任务自动的生成一个 初始策略，并且当一个新的信息到达是更新这个策略在其执行过程中。
>
>我个人认为，这个设计，是对于 问题2 “安全需求会随着用户任务和执行状态变化”的问题的一种解决，让LLM自适应的根据用户任务生成一个初始策略，然后策略的更新也是基于新信息。

Each proposed update is determined by an SMT solver to be either a narrowing (applied automatically) or an expansion (requiring explicit approval), ensuring that the agent’s effective action space can only shrink without approval (monotonic confinement)

This deterministic update mechanism preserves utility and prevents silent privilege escalation, even when adversarial inputs are present.

> 每一个被提出的更新都确定性的由SMT解算器 去区分，区分 这个更新是一个 缩小更新（自动应用）或者是一个扩展更新（要求明确的批准），这确保了Agent的有效行为空间在未经批准的条件下只能减小（单调）。这种确定性的更新机制保证了有用性，并且避免了静态权限提升，并且即使攻击信息注入情况下也能保持。
>
> 我个人认为，这个设计，是对于 问题3的解决，降低权限自动应用 保障了可用性实用性，升高权限需要明确批准 保障了安全性。

![002](./note_1.assets/002.png)

Our evaluation on popular benchmarks (i.e., AgentDojo and ASB) shows that Progent significantly reduces attack success rates while maintaining high utility. We further validate Progent’s practicality by showcasing its effectiveness in real-world agent frameworks such as LangChain and OpenAI Agents SDK.

> 这项工作在公认的基准测试上进行测试，并且Progent展现出了优秀的降低攻击成功率的能力，并且保持了高实用性。我们还评估了Progent的实用性，通过展现它在现实使用的Agent架构中的有效性，这些Agent框架 例如 LangChain 和 OpenAI Agents SDK。



### Introduction

**Attack Surface**

![image-20261020135941950](./note_2.assets/image-20261020135941950.png)



#### Security Risks in AI Agents

The ability to interact with external environments makes AI agents powerful, enabling them to perform diverse tasks across domains such as communication, scheduling, and finance. However, this same ability also creates a large attack surface. Recent research has raised serious security concerns about AI agents [15, 28, 52], where attackers exploit the agent’s interaction with the environment by injecting adversarial instructions into external data sources. When the agent retrieves such data via tool calls, the injected instructions can redirect the agent to perform dangerous actions, such as unauthorized financial transactions [11] or data exfiltration [29]. This class of attacks is known as *indirect prompt injection* [14, 30].

> Agent 与外部环境交互的能力成就了Agent的强大，这个能力使得Agent可以跨领域处理大量的任务，例如通信、调度和财务。然而，这个能力同样带来了一个巨大的攻击面。以往的研究已经对Agent提出了严重的安全担忧，攻击者通过在外部数据中注入攻击指令达到利用Agent与外部环境的交流的目的。当agent通过 工具调用来检索这种数据，这些注入指令可以重定向agent以达到执行危险操作的目的，例如非授权的金融交易或者数据外泄。这一类的攻击就是大家熟知的 *间接提示词注入*。

#### Challenges for Securing AI Agents

First, agent behavior is inherently non-deterministic: an LLM processes unstructured natural language and may make different decisions across runs. In contrast, security requires deterministic enforcement that holds regardless of how the LLM reasons, even under adversarial influence. Reconciling these
two worlds is non-trivial. 

> 首先，agent的行为是内在的不确定性的：一个LLM处理非结构化的自然语言并且会在运行过程中做出不同的决定。与之相对的是，安全需要确定性的严格执行不论是LLM在推理，甚至是在攻击者的影响下。很难去调和这两者。
>
> 我对这一点的理解是，Agent的执行过程是随机的、不确定的，但是 安全是要求 确定的，Agent执行过程中的不确定性与 安全要求的确定性之间的矛盾，是很难解决的。

Second, agent security depends on both the user’s task and the information the agent gathers during execution. Different tasks require different tools and arguments, and the relevant context evolves at runtime as the agent interacts with the environment. A defense must track these changes automatically, since involving the user in every security decision is impractical and negates the benefit of an autonomous agent. 

> 此外，Agent安全 同时依靠 用户的任务 和 Agent在执行过程中收集的信息。不同的任务需要不同的工具调用和参数，并且 在Agent与外界进行交互的时候，相关的上下文也会在运行时不断地变化。一个防御系统必须能够自动的追踪这些变化，并且让用户参与每一个安全决策是不切实际地，这破坏了自动化Agent地优势。
>
> 我对这一点的理解是，Agent的安全需要 不断地收集信息，而这些信息是不断变化的，Agent安全防护系统必须能够动态的捕获这些变化，并且 应自主的（或者部分自主的）去作出安全决策，所以我推断 这里的安全决策大概率是 全自动（完全由系统做出判断）或者是 半自动（人机协同）的，绝对不可能是全人工去做，这样会破坏Agent的使用体验。

Finally, agents must sometimes expand their privileges to complete benign tasks. Yet, adversaries also seek to exploit the same mechanism to induce
harmful actions, and both legitimate and adversarial expansion requests arrive through the same channel. Distinguishing legitimate from adversarial expansions, without sacrificing security or utility, is a fundamental challenge. 

> 最后，Agent时而必须去扩展他们的权限 去完成一些正常的任务。然而，攻击者也去利用这个机制去诱发攻击行为，并且 合法的和攻击的扩展请求往往会在相同的通道到达。区分合法的和攻击的扩展请求，并且不牺牲安全性和实用性，是一个根本性的挑战。
> 我对这一点的理解是，Agent的安全防护系统 还必须能够精准地区分 正常 和 攻击的权限扩展请求，并且保障 安全性和可用性。

> 对于以上三点挑战，应该就是本文要解决的.三个问题。
>
> 1. Agent的执行是不确定的，安全是要求确定性的，安全系统如何在Agent执行过程中，保障安全的确定性？
> 2. Agent的安全判断所需要的信息是动态的，安全系统如何获取动态的信息，并且尽可能自主的做出判断？
> 3. Agent的正常权限扩展和攻击权限扩展如何进行分辨？















