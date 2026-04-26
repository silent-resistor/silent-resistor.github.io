---
title: "SDLC"
date: 2026-04-14
---

## 1. What is SDLC?
- The Software Development Lifecycle is a set of practices that make up a framework to standardise building software applications. SDLC defines the tasks to perform at each software development stage. This methodology aims to improve the quality of the software and the development process to exceed customer expectations and meet deadlines and cost estimates. 
- The aim of SDLC is to establish repeatable processes and predictable outcomes from which future projects can benefit. SDLC phases are usually divided into 6-8 phases.
    - **Planning**: The planning phase encompasses all aspects of project and product management. This includes resource allocation, project scheduling, cost estimation etc.
    - **Requirements Definition**: is considered part of planning to determine what the application is supposed to do and its requirements.
    - **Design & Prototyping**: establishing how the software application will work, programming language, methods to communicate with each other, architecture etc
    - **Software Development**: entails building the program, writing code and documentation.
    - **Testing**: In this phase, we ensure components work and can interact with each other.
    - **Deployment**: in this stage, the application or project is made available to users.
    - **Operations & Maintenance**: this is where engineers respond to issues in the application or bugs and flaws reported by users and sometimes plan additional features in future releases.


## 2. What is Secure Software Development Lifecycle (SSDLC) ?
- During SDLC, security testing was introduced very late in the lifecycle. ﻿Bugs, flaws, and other vulnerabilities were identified late, making them far more expensive and time-consuming to fix. In most cases, security testing was not considered during the testing phase, so end-users reported bugs after deployment. Secure SDLC models aim to introduce security at every stage of the SDLC.
- A study conducted by the Systems and Sciences institute at IBM discovered that it costs six times more to fix a bug found during implementation than one identified as early as during the design phase. It also reported that it costs 15 times more if flaws are identified during testing and up to 100 times more costly if identified during the maintenance and operation phases. 
### 2.1 Understanding security posture
- Perform gap analysis to identify what activities and policies exist in your organisation and how effective they are.
- Create software security intiatives (SSI) by establishing realistic and achievable goals with defined metrics for success.
- Formalise processes for security activities within your SSI.
- Invest in security training for engineers as well for appropriate tools.
### 2.2 SSLDC Processes
- Generally speaking, a secure SDLC involves integrating processes like security testing and other activities into an existing development process. The following are some of the key processes:
    - **Risk Assessment** - during the early stages of SDLC, it is essential to identify security considerations that promote a security by design approach when functional requirements are gathered in the planning and requirements stages. 
      - **Qualitative Risk Assessment** (subjective like high, low etc). Risk formula, Risk = Severity x Likelihood
      - **Quantitative Risk Assessment** (numbers)
    - **Threat Modelling** - is the process of identifying potential threats when there is a lack of appropriate safeguards. It is very effective when following a risk assessment and during the design stage of the SDLC, as Threat Modelling focuses on what should not happen. In contrast, design requirements state how the software will behave and interact.
      - STRIDE, DREAD, and PASTA are among the common threat modelling methodologies.
      - **STRIDE** stands for Spoofing, Tampering, Repudiation, Information Disclosure, Denial Of Service, and Elevation/Escalation of Privilege. We can use STRIDE to identify threats, including the property violated by the threat and definition.
      - **DREAD** stands for five questions about each potential threat: Damage Potential, Reproducibility, Exploitability, Affected Users, and Discoverability. It ranks threats by assigning scores based on their severity and risk probability. 
      - **PASTA** stands for Process for Attack Simulation and Threat Analysis. It aligns technical requirements with business objectives and provides a strategic view of threats. 
    - **Code Scanning / Review** - Code reviews can be either manual or automated, leveraging Static and Dynamic Security testing technologies crucial in Development stages. 
      - Static analysis examines source code without executing programs, while Dynamic analysis looks at source code during runtime. Static analysis detects bugs at implementation level, while dynamic analysis detects errors during program runtime. 
      - **SAST** (Static Application Security Testing) is a white box method used to scan source code for security vulnerabilities that could provide backdoors for attackers.
      - **Software Composition Analysis (SCA)** is used to scan dependencies for security vulnerabilities, helping development teams track and identify any open-source component brought into a project. SCA is now an essential pillar in security testing as modern applications are increasingly composed of open-source code.
      - **DAST** (Dynamic Application Security Testing), a black-box testing method that finds vulnerabilities at runtime in applications, mostly webapps in production deployments, and send alerts to security teams.
        - DAST works by simulating automated attacks on an application, mimicking a malicious attacker. The goal is to find unexpected outcomes or results that attackers could use to compromise an application.
      - **IAST** (Interactive Application Security Testing), analyses code for security vulnerabilities while the app is running. It is usually deployed side by side with the main application on the application server. It is an application security tool designed for web and mobile applications to detect and report issues evn while running. 
      - Before someone can fully comprehend IAST's understanding, the person must know what SAST and DAST mean. IAST was developed to stop all the limitations in both SAST and DAST. It uses the **Grey Box Testing** Methodology.
      - **RASP** (Runtime Application Self Protection) is a runtime application integrated into an application to analyse inward and outward traffic and end-user behavioural patterns to prevent security attacks. This tool is different from the other tools as RASP is used after product release, making it a more security-focused tool when compared to the others that are known for testing.
    - **Security Assessments** - Penetration Testing and Vulnerability Assessments are forms of automated testing that identify critical paths in applications that may lead to exploitation of vulnerabilities.  
      - **Vulnerability Assessments** focus on Finding Vulnerabilities, but do not validate them or simulate the findings to prove they are exploitable in reality. Typically, automated tools run against an organisation's network and systems. Examples of tools: are OpenVAS, Nessus (Tenable), and ISS Scanner. The result is a report with a list of vulnerabilities usually found, with an automated threat level severity classification
      - **Penetration Testing** includes Vulnerability Testing but goes more in-depth. It is extended by testing/validating of vulnerabilities, quantifying risks and attempting to penetrate systems. For example, trying to escalate privileges after a vulnerability is found, some vulnerabilities can be a lower risk but can be used as leverage to cause more damage




## 3. Pipelines Notes
- Git never forgets, if developers commit sensitive information like API keys, passwords, or tokens, they will be available in the commit history forever unless removed.
- So tools like [GittyLeaks](https://github.com/zricethezav/gitleaks), which would scan through the commits for sensitive information.
- Log4j Zero day effected software list - https://github.com/cisagov/log4j-affected-db/tree/develop/software_lists
- In modern pipelines, unit testing can be used as quality gates. Test cases can be integrated into the Continuous Integration and Continuous Deployment (CI/CD) part of the pipeline, where the build will be stopped from progressing if these test cases fail. However, unit testing is usually focused on functionality and not security.







