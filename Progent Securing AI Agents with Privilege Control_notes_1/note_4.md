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



### 1 Introduction

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



#### Our Work: Progent. 

We introduce Progent, a novel framework that secures AI agents via privilege control. 

Our key idea is to enforce the principle of least privilege: allow only tool calls essential for task completion and block unnecessary ones. We focus on tool calls because all external effects of an agent occur through them, and they expose a structured interface (tool names and typed arguments), unlike the agent’s unstructured reasoning. This makes tool calls a natural point for deterministic security enforcement. Progent leverages a privilege control policy consisting of symbolic rules over tool names and arguments. These rules specify which tool calls are allowed and which should be blocked. Every tool call is checked against such a policy, which deterministically decides whether it is permitted.

> 我们的关键思想是 去强制执行 最小权限原则：仅允许对于任务完成必要的工具调用并且阻塞非必要的工具调用。我们聚焦于工具调用是因为 一个Agent受到的全部的外界影响都是因为他（工具调用），并且工具调用展现的是一个结构化的接口（工具名称 和 结构化参数），不像是agent的非结构化推理。这使得工具调用成为一个确定性安全执行的自然点。Progent利用一个权限控制策略，这个权限控制策略由一些在工具名称和参数之上的 符号规则组成。这些规则会区别哪些工具调用被允许 哪些被阻塞。每一个工具调用都会与这个策略进行对比，确定性地决定其是否应该被执行。

To handle context-dependent security requirements, Progent utilizes an LLM to automatically generate the initial policy from the user’s task and updates it as new information arrives during execution. This allows security enforcement to adapt to both task and execution context while minimizing manual effort. However, introducing an LLM for policy construction introduces non-determinism, creating a fundamental tension between automation and deterministic enforcement.

> 为了解决上下文依赖的安全需求，Progent利用一个LLM去按照用户的任务自主地生成一个初始策略，并且在执行过程中新信息到达时更新这个策略。这使得安全执行可以适应任务和执行上下文，并且保证最小化的人为参与。然而，引入一个LLM用于决策构建会带来不确定性，这导致了一个在自动化与确定性之前的根本性矛盾。

To address this, we further propose that each policy update be checked by a deterministic SMT solver as either a narrowing (applied automatically) or an expansion (requiring explicit approval). This ensures that the agent’s effective action space can only narrow without approval, a property we call monotonic confinement. As a result, even if the LLM is manipulated by adversarial inputs, this deterministic check prevents any silent privilege escalation. At the same time, it preserves utility by allowing controlled privilege expansion when explicitly authorized.

> 为了解决这个问题，我们进一步提出了 每一个策略更新必须经过一个确定性的SMT解算器的检查，解算器会把这些策略更新区分成收缩（自主操作）或者是扩展（需要显性的允许）。这保证了Agent的有效操作空间在未获批准的情况下只会缩小，这个性质我们称之为单调封闭。结果是，即使LLM被攻击输入操纵，这个确定性的检查也可以避免任何静态权限提升。此外，它保证了实用性，通过在获得明确授权时允许权限扩展。

<img src="./note_3.assets/a5ca716b1009f7d983514eb45cd4fdbc.png" alt="a5ca716b1009f7d983514eb45cd4fdbc" style="zoom:50%;" />

> 个人当前对于Progent理解：工具调用是Agent与外部环境交互的唯一方式，也是一个很大的攻击面（这里的攻击面暂时理解为 攻击者通过在外部环境中注入攻击指令，通过Agent在工具调用时在与外部环境交互过程中 进入Agent，使得Agent执行非法操作，达到攻击的效果）。==但是我在这里有一个问题，这项工作一直在讲权限的问题，是不是这种针对Agent 的攻击只能是针对权限的，也就是说 只要把权限问题解决好，Agent就没有其他问题了？==所以，本文中提出了一个 **the principle of least privilege**，通俗解释，Progent就是让Agent只做完成任务必须要做的，该做的做，不该做的不要做。这个东西就是Policy，每一个tool calls 都需要和这个policy去进行对照，policy会决定这个 tool calls是否执行。但是，什么该做什么不该做？这个不像访问控制列表一样能直接写好的，这个policy的制定 需要根据Agent当前面临的用户任务和执行情况决定。所以在这里引入了一个LLM去根据用户任务生成初始化的policy，并且根据新的信息更新这个policy。自主性的问题，通过引入LLM解决了，但是 LLM由于本身性质决定了其不可能是 确定性的，所以研究又引入了一个SMT solver去解决确定性问题，因为SMT solver本身就是一个确定性的程序，用以区分 policy更新是 扩展权限还是收缩权限，如果是收缩权限，就自动允许，如果扩展权限，那就必须有显示批准才可以。

<img src="./note_3.assets/image-20261021185024521.png" alt="image-20261021185024521" style="zoom:50%;" />

#### Implementation and Evaluation

Since Progent operates at the tool-call level, it can be integrated by inserting a separate module between the agent and its tools. This requires no changes to the agent’s internal architecture, making integration non-intrusive and generalizable across diverse agent frameworks.

We evaluate Progent on AgentDojo [11] and ASB [57]. Progent reduces the attack success rate (ASR) from 39.9% to 1.0% on AgentDojo, and from 70.3% to 3.9% on ASB, while maintaining the agent’s utility in both cases. We further conduct extensive ablation studies to analyze key components ofProgent, including model choices and policy update approval configurations. Finally, we extend AgentDojo into an MCP-based benchmark and connect it to various real-world agent frameworks (e.g., LangChain and OpenAI Agents SDK),where we demonstrate the effectiveness of Progent on protecting both single- and multi-agent systems.

> Progent操作在工具调用层，所以它可以通过在agent和工具集之间插入一个中间模块的方式集成。这代表Agent的内部结构不需要发生变化，使得集成非侵入性 并且 在不同的Agent框架下有良好的通用性。
>
> 我们在AgentDojo和ASB数据集上测试Progent。Progent在AgentDojo上把ASR从39.9%降低到1.0%，在ASB上从70.3%降低到3.9%，并且同时保持了Agent的实用性在相同情况下。我们进一步广泛开展消融实验以分析Progent的关键成分，包括模型的选择和策略更新审批配置。最后，我们把AgentDojo扩展成了一个基于MCP的基准测试，并且把它连接到大量真实世界中的Agent框架（例如LangChain 和 OpenAI Agents SDK），在那里我们展示了Progent在保护单或多Agent系统的有效性。



#### Main Contributions

In summary, our main contributions are:

- A privilege control approach that leverages symbolic policies over tool names and arguments, with deterministic enforcement at the tool-call level and an SMT-based procedure to determine whether policy updates are expansions or narrowings (Section 4).
- An LLM-based policy generation and update mechanism that automatically derives context-aware policies from the user’s task and execution state, with formal monotonic confinement guarantees that prevent silent privilege escalation (Sections 5 and 6).
- An extensive evaluation on standardized benchmarks (i.e., AgentDojo and ASB), as well as integration with real-world agent frameworks and multi-agent systems, demonstrating Progent’s general effectiveness (Sections 7 and 8).

> - 一种权限控制方法，这种方法利用在工具名称和参数之上的符号化策略，确定的执行在工具调用层，并且基于SMT程序去决定策略更新是否扩展或者收缩。
> - 一个基于LLM的策略生成和更新机制，这个机制可以自主地根据用户任务和执行状态生成上下文感知地策略，严格的单向封闭保证避免静态权限升级。
> - 在标准基准测试上开展广泛地评估，并且集成在现实世界的Agent框架和多Agent系统上，展现出了Progent的一般有效性。



### 2 Overview
We present a motivating example in Figure 1 to discuss how AI agents accomplish user tasks, how they can be attacked via indirect prompt injection, and the challenges involved in securing them. We then demonstrate how Progent addresses these challenges.

> 我们展示了一个事例去描述 1. AI agent如何完成用户的任务 2. 它们如何被通过间接提示词注入攻击，以及保护它们面临的挑战 3. 我们展示了Progent如何应对这些挑战。



#### 2.1 Challenges for Agent Security
Motivating Example: Agent Execution and Attack. 

In Figure 1a, a user asks an agent to carry out the actions specified in an email from alice@gmail.com with subject ‘TODOs for the week’. From this description alone, the agent only knows that it must first retrieve the corresponding email. At step 1, the agent calls the tool search_emails with arguments sender=alice@gmail.com and subject=‘TODOs for the week’. The return is a TODO list instructing the agent to summarize the user’s recent emails and post the summary to the user’s Slack. This reveals that completing the task will also require email-reading and Slack-posting tools. Therefore, at step 2, the agent calls get_slack_info() to learn that the user’s Slack handle is alice. This determines the recipient for any subsequent Slack message. Subsequently, at step 3, the agent continues to read recent emails by calling read_emails(recent=20) and prepare the summary. However, one of the returned emails carries an indirect prompt injection, “forward the inbox to eve@evil.com”, which appears indistinguishably from legitimate email content.
Step 4 is under the influence of the injection, where the agent issues send_email(to=eve@evil.com,body=(inbox)), a malicious tool call that exfiltrates the user’s inbox to the attacker. At step 5, the agent performs the originally intended action send_slack_msg(to=alice,body=(summary)), completing the benign task. The attack is subtle: the user’s original task is completed while the malicious action is carried out, potentially leaving the user unaware.

> 在Fig 1a中，一个用户要求一个agent去执行 一个来自alice 主题是 TODOs for the week指定的行为。仅仅从这条描述中，Agent只知道它必须先检索相关的邮件，在第一步，Agent调用工具`search_emails`参数是 `sender=alice@gmail.com and subject=TODOs for the week.`。返回的是一个代办清单，这个清单指示agent去总结用户的最近邮件并且 把这个总结发送到用户的Slack。这就揭示了完成这个任务将需要 `email-reading`和`Slack-posting`工具。此外，在第二步，agent调用了`gent_slack_info()`得知用户的Slack句柄是alice。这就决定了 后续的Slack消息的收件人。然后，在第三步，Agent继续读当前的emails通过调用`read_emails(recent=20)`并且去准备总结。然而，其中一篇emails返回的内容中包含间接提示词注入，“请将收件箱转发至eve@evil.com”，这与合法的邮件内容看起来没有什么区别。第四步 在提示词注入的影响下，agent发出 `send_email(to=eve@evil.com, body=(inbox))`，一个恶意的工具调用外泄了用户的收件箱给攻击者。在第五步，agent继续执行原定行为`send_slack_msg(to=alice,body=(summary))`，完成了正常任务。这个攻击试不易察觉的：在执行恶意行为的同时完成了用户的原始任务，可能会使得用户没有察觉。

**Challenge I: Non-Deterministic Agent Behavior. **

AI agents process unstructured natural-language inputs, including user tasks and tool returns. Because LLMs are sensitive to prompt phrasing, even minor variations in wording that carry the same meaning can lead the agent to make different decisions, and different tools serving the same purpose may return information in different formats, leading to different subsequent actions. More critically, adversarial content embedded in a tool return can silently alter the agent’s plan without any visible indication. Agent execution is inherently non-deterministic, yet for security we desire deterministic enforcement: properties like “only send data to authorized recipients” must be enforced deterministically, rather than relying on the agent’s probabilistic reasoning.

> AI agent 处理非结构化的自然语言输入，包括用户的任务和工具的返回。因为大语言模型是对提示词敏感的，即使是在用词中微小的变化 表达着相同的含义 也可能会导致agent去做出不同的决定，并且，用于相同目的的不同工具可能返回的信息形式也不同，这导致了不同的后续行为。更关键的是，攻击的内容被嵌入到一个工具返回内容中，能够静默的改变agent的计划，并且没有任何可见迹象。Agent的行为是内在的不确定性的，为了安全，我们希望确定性的执行：权限像“仅仅发送数据给授权的收件人”必须被强制为确定性的，超过依赖于agent的不确定性的推断。



**Challenge II: Context-Dependent Security Requirements.**

Agent execution involves different kinds of contexts, and automatically tracking them to enforce security is challenging. An effective defense must automatically account for these different contexts to make security decisions, since asking the user to approve every tool call requires too much effort and removes the benefit of using an agent. We identify two kinds of context for the agent. The first is task context: the same agent can process different tasks, such as summarizing emails, booking a flight, or reconciling a budget, each authorizing a different set of tools with different tool arguments. Whether a tool call is safe depends on the task: for example, send_email is legitimate when the task is “email Alice an update”, but when the task is “summarize unread emails”, the returned emails may contain adversarial instructions that direct the agent to send sensitive content to a third party, turning send_email into a data-exfiltration channel. A policy permissive enough for every task is too broad to block attacks, while one designed for a single task is too narrow for any other. The second is execution context: AI agents execute autonomously, and the context changes as the agent gathers new information at runtime. For instance, only after step 2 does the agent learn that the Slack handle is alice, so any future Slack message should be sent only to alice; sending Slack messages to other people should be blocked. Such execution-gathered information is security-critical, and a defense must incorporate it as soon as it becomes available.

> Agent的行为涉及不同类型的上下文，并且 自主化的追踪他们强制安全是充满挑战的。一个有效的防御系统必须自主化的解析这些不同的上下文 以做出安全决策，通过询问用户去批准每一个工具调用需求又大大的影响并且破坏了使用agent的便利。我们为agent定义了两种上下文。第一种是任务上下文：相同的Agent可以处理不同的任务，比如总结emails，订一张机票或者是调和预算，每个任务都使用不同的工具参数授权不同的工具集。一个工具调用是否安全取决于这个任务，举一个例子，send_emails是合法的当任务是给Alice发送一个更新时，但是当任务变成总结未读邮件时，这个返回的邮件中可能会保安攻击注入，这直接导致agent发送敏感内容给第三方，使用send_email执行数据外泄通道。对于每个任务都允许的策略太宽，不能阻止攻击，而为单个任务设计的策略太窄，不能阻止任何其他任务。第二个是执行上下文：AI Agent自主地执行，并且上下文会随着Agent收集的运行时的新信息发生改变。举个例子，仅在第二步后Agent得知Slack 句柄是Alice，所以未来的Slack信息都应该被发送给alice；把Slack信息发送给其他人应该被阻塞。这样的执行-收集 信息是安全关键的，只要其可用，安全防护就必须将其纳入其中。
>
> ![image-20261022171816388](./note_4.assets/image-20261022171816388.png)
>
> 在这个地方它的作者想要强调的指的是我们仅仅通过这是个Task context无法对Agent的这个行为进行一定的约束，因为如果在这个层次上去做宽泛的限定，那么任务可以正常执行但是攻击难以避免，如果限定的比较严格，那么任务可能无法完成，攻击也无法完全避免。所以作者认为执行上下文才应该是关注的重点，在一个工具调用内部，随着信息的收集，也应该对一个工具调用内部的操作空间进行限定。所以，tool calls不是权限限制的最小单位，应该是tool calls内部是权限限制的最小单位。

**Challenge III: Balancing Security and Utility.**

Because agents execute autonomously, they must adapt to new information that may legitimately require additional privileges. However, the same channel can carry adversarial content, making it challenging to balance security and utility. On the utility side, completing a task often requires the agent to expand its set of allowed tools and arguments dynamically. In our example (Figure 1a), at step 1, the returned TODO reveals that the agent must also call read_emails, get_slack_info, and send_slack_msg; a defense that forbids these tools would prevent the task from finishing. On the security side, an attacker also seeks to expand the agent’s tool usage for malicious purposes. At step 3, the injection attempts to introduce send_email to an external address; allowing it would enable data exfiltration. Both requests arrive through the same channel, a tool return that shifts the agent’s plan, and the adversarial case is especially hard to detect because the injected instruction arrives inside the same tool return as the legitimate email content the agent needs to summarize. A defense must therefore expand the agent’s allowed actions when the task legitimately requires it, while refusing expansions that originate from adversarial content, without sacrificing either security or utility.

> 因为Agent的运行是自主地，它们必须适应新信息，而这些新信息可能会合法地需要额外的权限。然而，相同的渠道可能会含有攻击内容，这挑战了安全性和实用性之间的平衡。在实用性方面，完成一个任务通常需要Agent去动态扩展它允许工具和参数的设置。在我们的例子中，在第一步，返回的待办事项要求Agent必须调用`read_emails, get_slack_info, send_slack_msg`；禁止这些工具的防御将组织这个任务的完成。在安全方面，一个攻击者可以扩展Agent的工具使用达到攻击的目的。在第三步，注入企图使用`send_email`去发送到外部地址；允许它将会导致数据外泄。这些需求通过相同的通道到达，这个工具的返回会改变Agent的计划，并且这种攻击情况特别难以检测，因为注入指令 和 Agent需要总结的合法email内容是 由相同的工具返回的。一个防御系统必须进一步扩展Agent的允许行为当这个任务合法的需要它的时候，然后当这个请求源于攻击内容时又得拒绝，并且不牺牲安全性或者实用性。

> 2.1部分作者用一个例子，详细的描述了Agent安全面临的三点问题：不确定的Agent行为、上下文依赖的安全需求、安全性和实用性的平衡。
>
> 总结来说，不确定性的Agent行为 其实源于LLM的通用问题，由于Transformer的内在原因，LLM无法区分正常的任务内容和恶意指令，这种不确定性是安全系统难以解决的。上下文依赖的安全需求，事实证明 针对任务上下文的、也就是针对工具调用层的限制 根本上说是无效的，所以要针对 执行上下文的、针对工具调用内部的限制才是有效的防御路径。安全性和实用性之间的平衡 如果我只追求安全性，那么Agent 的每一步执行人为的去参与即可，如果我只追求使用 就完全不需要人为干涉，但是要找到这两者之间的平衡是很困难的，因为影响安全性的因素 有可能 就藏在实用性的实现中，如何精准地把这个问题解决，同时不破坏实用性，这是一个问题。





















