---
tags:
  - career
  - english
  - interview
---

## Q1: "Could you tell me a little bit about yourself and a recent project you've been working on?"


### The Critique

- **"In software industries":** Usually, we say "the software industry" (singular) or "the tech industry."
- **"In a daily basis":** The correct preposition is **on** a daily basis.
- **"Firstly... Firstly":** You used this twice to start sentences. We can vary this with "Originally," "Initially," or "Early in my career."
- **"Before long":** This is okay, but "Later," or "Subsequently" sounds more professional in an interview context.
- **Game Boy Project:** This is a great "passion project" to mention! We just need to connect it more clearly to your professional growth.


### Adjusted

> "Nice to meet you. My name is Jacky, and I’ve been in the **software industry** for nearly a decade. I **initially** started as a Backend and DevOps engineer, where I developed APIs and managed VM and server resources **on a daily basis**.
> 
> When I joined Trend Micro, my focus shifted toward **low-level development** in C and C++. I spent time with the ERS team working on email protection before moving to the Orca team. There, I specialized in developing **Linux kernel modules and eBPF programs**.
> 
> Most recently, I’ve been building a **Game Boy emulator** as a personal project. It’s been a fantastic way to deepen my understanding of **computer architecture**, specifically in areas like **instruction decoding, memory banking, and interrupt handling**."


---

## Q2: "That’s an impressive transition into systems programming. Since you mentioned working on the Orca team, **could you describe a specific technical challenge you faced while developing Linux kernel modules or eBPF programs, and how you resolved it?**"


## Answer

> I think the most impressive technical challenge that I can recall is when I was doing the powerpc ISA support. My mission was to further extend our linux kernel to be able to run on any powerpc machine. 
> I faced lots of challenges, including writing powerpc assembly, writing kernel code for page walking, and the most difficult part is the TOC (table of content) problem. The problem is caused by an unique mechanism in powerpc, powerpc use `r2` (General Purpose Register 2) as an memory location to access the so-called table, but the table is actually different when you are in kernel module than in the Linux kernel itself. So it leads to kernel crashes. To mitigate this problem, I do a lots of research and use crash tools to debug. On the same time, discussed with my colleagues for different point of views.

### The Critique

- **"Doing the powerpc ISA support":** In an interview, it's better to say "implementing support for" or "porting our module to."
- **"Further extend our linux kernel":** Usually, we say "extending our software's compatibility" or "porting our kernel-level security agent."
- **"I do a lots of research":** Use "I conducted extensive research."

### Key Vocabulary

- **Hurdle:** A more professional word for "difficulty" or "problem."
- **Deep dive:** A great phrase to show you did thorough research.
- **Mismatch:** A perfect word for when two memory locations or configurations don't align.
- **Cross-module calls:** This adds technical precision to your explanation.

## Adjusted

> "The most significant technical challenge I recall was **implementing PowerPC ISA support** for our security agent. My mission was to ensure our Linux kernel module could run seamlessly across different PowerPC environments.
The primary hurdle was the **TOC (Table of Contents)** mechanism unique to the PowerPC architecture. In this architecture, the **r2 register** points to a table used for global data access. However, a conflict arises because the TOC for a **loadable kernel module** is different from the TOC used by the **base kernel itself**. This mismatch led to immediate system crashes when the module attempted to access global symbols.
To resolve this, I **conducted a deep dive** into the PowerPC ABI and utilized **crash tools** to analyze the register states during the panic. After collaborating with my team and researching how the kernel handles module linkage, I implemented a solution to **correctly manage the r2 register state** during cross-module calls. This successfully stabilized the module and allowed us to expand our platform support to PowerPC architectures."

---

## Q3: Can you tell me about a time you had a disagreement with a teammate or your lead regarding a technical decision? How did you handle it?

## Answer

> Actually, this happened quite often. Our technical lead  always had lots of ideas and he always came up with technical solutions, but I'm also the kind of person who is always seeking for the best solutions. I enjoy the process of both seeking for good solutions and writing code to make it come true. When there is a disagreement between us, I will need to persuade him about my solution. Sometimes he will find out potential bugs or come up with better idea during the discussion. Sometimes I'll need to conduct extensive research to make my solution complete. So this is actually the process of debate and persuading each other.

### The Critique

- **"Actually, this happened quite often":** While honest, starting this way can sometimes sound like there is constant friction. It’s better to phrase it as "In a high-performing technical environment, healthy debate is common."
- **"Persuade him":** A better professional term is "presenting a case" or "advocating for a solution."
- **"Make it come true":** This is a bit poetic. In a technical context, "implementing" or "realizing the solution" sounds better.
- **"He will find out potential bugs":** This is a great point to include! It shows humility and that you value his feedback.

### Key Vocabulary Improvements

- **Optimal solution:** A professional way to say "the best solution."
- **Advocating:** To support or argue for a specific cause or policy.
- **Proof-of-concept (PoC):** A small-scale demonstration used to prove that a certain method or idea is feasible.
- **Stress test:** Using a situation to see how well a system (or in this case, an idea) stands up under pressure.
- **Robust:** Sturdy and unlikely to fail.

## Adjusted

> "I believe that healthy technical debate is essential for a high-performing team. My technical lead and I both prioritize finding the **optimal solution**, though we occasionally approach problems from different angles.
When our perspectives differ, I focus on **advocating for my solution using data and research**. I often conduct a deep dive into the documentation or create a proof-of-concept to demonstrate the feasibility of my approach. However, I also view these discussions as a **collaborative filter**. There are times when his feedback uncovers edge cases or potential bugs I hadn't considered, and other times where my research proves the stability of my proposal.
Ultimately, we treat these disagreements as a 'stress test' for our architecture. By the end of the conversation, we aren't just choosing 'my' way or 'his' way—we are settling on the **most robust solution** for the project."

---

##  "That’s a very mature way to handle technical friction. Now, looking at your career path from DevOps to Kernel development: **Where do you see yourself in the next three to five years, and how does this role fit into that vision?**"

## Answer

> As a software engineer, it's always necessary to keep up with the trend. I think the next 3-5 years will still be the age of AI, and we need to quip ourselves with the power of AI, because merely output runnable code is not enough now. We need to leverage AI to quickly understand the structure of a code base , and we can generate unit tests and documents easily with AI. Without AI's assistance, the process of development is too slow, it feels like coming back to stone age. But the foundation of AI is still computer resources and chips, I think working in a upstream industry of chip manufacturing is somewhere I can stand in the AI trend and will not be flush away by it.


### The Critique

- **"Quip ourselves":** The word you are looking for is **"equip"** (to provide with what is needed). "Quip" means to make a witty remark.
- **"Merely output runnable code":** A more professional way to say this is **"simply producing functional code."**
- **"Flush away":** The common idiom is to be **"washed away"** or **"left behind."**
- **"Upstream industry":** This is a great term. You can also use **"lower levels of the stack"** or **"the hardware-software interface."**
- **"Stone age":** While descriptive, in an interview you might say "it feels inefficient compared to modern standards."

### ### Key Vocabulary Improvements

- **Development Lifecycle:** A professional way to describe the "process of development."
- **Mapping a Codebase:** A common term for using tools to understand how a large project is structured.
- **Semiconductor Innovation:** A more formal way to discuss "chips."
- **Performant and Optimized:** These are "power words" for systems engineers that indicate you care about speed and efficiency.
- **Hardware-Software Interface:** A sophisticated way to describe the area where kernel and driver work happens.

## Adjusted

> "I believe the next three to five years will continue to be defined by the AI revolution. In this environment, **simply producing functional code is no longer enough**; we must **equip ourselves** with AI to stay competitive. I see AI as a powerful tool for rapidly mapping complex codebases, generating comprehensive unit tests, and automating documentation. Without these tools, the development lifecycle feels significantly less efficient.
However, the foundation of AI ultimately rests on **compute resources and semiconductor innovation**. This is why I am so focused on low-level systems and kernel development. By working in the **upstream of the industry**—closer to the hardware and chip architecture—I can position myself at the core of the AI trend. My goal is to ensure that the software and kernels driving these AI innovations are as **performant and optimized** as possible, ensuring I remain at the forefront of where the most critical technical challenges exist."