# Introduction

This lab is divided into three parts. In each part, you will experiment with various aspects of software Isolation using Docker, a popular platform for packaging applications and their dependencies into lightweight, portable containers, each in its own isolated environment[^P11]. 

Each of the different parts are intended to be roughly a practical (PL) class, but your mileage may vary; feel free to do it at your own pace, but don’t forget to be thorough on your observations and report. Each part has a setup section indicating what you will need to perform the lab.

Submit a lab report that describes the procedures you followed. Please explain noteworthy or unexpected observations. Don’t forget to address all the questions (in bold) in this lab description. Provide details on how you arrived at the conclusions/results (do not just paste the result). The lab report can be developed in a team of up to 4 members (exceptions must be approved). See the lab report submission dates in Moodle. Package your submission as described in the **Submission integrity & anti-fabrication** section below, and use the Docker image tag assigned to your team in Moodle throughout Part 1.

## Criteria

The Lab will be graded using the following rubric. Please use this detailed rubric as guidance to create a good, detailed report. If the assignment is deemed unacceptable in a given criterion, a 0 will be given to that respective criterion. A missing, fabricated, or mismatched proof-of-work artifact (see **Submission integrity & anti-fabrication** below) scores 0 for the corresponding question.

| **Criteria**                   | **weight**      | **Exemplary**  **(100%)**                                                                                                                                                              | **Accomplished**  **(75%)**                                                                                                                                   | **Satisfactory**   **(50%)**                                                                                                            | **Poor**  **(25%)**                                                                                                                           |
| ------------------------------ | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Organization and  Language** | 10%             | Good organization, easy to follow, and related to the lab activities.  No major language  (Grammar, Usage, Mechanics, Spelling) errors.                                                | Organized, but points are jumpy or hard to follow.  There are only one or two important language errors.                                                      | Some organization; points  jump around; hard to follow.  More than two important  errors.                                               | Poorly organized; no logical progression; hard to connect with lab activities and logical flow.  Numerous errors distract from understanding. |
| **Questions 1-15**             | 80%  (~6% each) | Complete answer  touching all the relevant points, with no or very minor incorrections. Artifacts (videos, images) requested are included.                                                                                                | Most relevant points were covered, but there were some small incorrections. Artifacts (videos, images) requested are included.                                                                                 | Does not address most key points. There are some major incorrections. Missing artifacts (videos, images).                                                                   | Unable to provide  specific details. Mostly incorrect details/facts. Missing artifacts (videos, images).                                                                          |
| **Question 16**                | 10%  (extra)    | Supporting details  specific to the exploit; Support information and explanations are  consistently accurate.  Demonstration fully  working.                                           | Some details are  non-supporting or marginal to the exploit, or some facts are inaccurate or  inconsistent. Demonstration fully working or with minor issues. | Details and/or facts are somewhat sketchy; Do not support exploit, or very marginally.  No working demonstration or significant issues. | Unable to provide  specific details. No demonstration or significant issues with it.                                                          |
| **Question 17**                | 10%             | Complete list of good practices covering the most relevant aspects (e.g., image security, containers, host,  service). Support information and explanations are consistently accurate. | Mostly complete; No more than a couple of points missing or incomplete. Some supporting information  missing                                                  | Some relevant points  missing and/or a lot of supporting information missing.                                                           | Unable to provide specific details.                                                                                                           |

This rubric produces the **team** report grade. Each member's final mark for the lab is that grade multiplied by their individual **discussion factor** (see the **Discussion** section).

## Discussion

Each lab includes an individual discussion during lab classes, where every group member must
demonstrate understanding of the work delivered. The rubric produces the **team** report grade;
each member's final mark for the lab is that team grade multiplied by their own **discussion
factor**:

| Level | Factor | What it looks like |
| ----- | ------ | ------------------ |
| Independent mastery | 1.00 | Explains the work and the reasoning (the *why*), answers follow-up questions, and can re-derive a step live on your own board/tag. |
| Solid | 0.85 | Explains the work and most of the reasoning; minor gaps are closed with light prompting. |
| Partial | 0.70 | Describes *what* was done but is shallow on *why*; needs significant prompting. |
| Superficial | 0.50 | Recites the report but cannot answer data-bound or "why" questions. |
| None | 0.00 | Absent without justification, silent, or clearly did not do or understand the work. |

Notes:

- The factor is individual, so members of the same team can receive different final marks.
- Questions are drawn from across all questions, not only the parts you personally wrote, so be ready to explain any of them.
- Expect questions bound to your own tag/host (the "Answer this" prompts in the artifact table) and possibly a request to reproduce a step live.
- Because each lab requires at least 9.5/20, a low discussion factor can drop a member below the pass threshold for that lab.


## Submission integrity & anti-fabrication

Each team is assigned its **own** Docker image tag in Moodle (e.g. `isepdei/insecurelabs01:team07`), which bakes secrets unique to your team. Your report must therefore be backed by evidence produced *from your own assigned tag and machine*, not by a plausible-sounding narrative. Three rules apply:

1. **Raw data before conclusions.** Any recovered secret, capability dump, or host-compromise claim must be accompanied by the raw artifact it came from (terminal transcript, extracted file, screenshot, or video) placed under `TARGETS/`. A conclusion without its reproducible source artifact scores **0** for that question.
2. **Per-question proof-of-work artifact.** The questions in the table below each require the listed artifact under `TARGETS/target-<n>/`. **A question whose artifact is missing, fabricated, or does not match your assigned tag / host scores 0, regardless of how well the narrative is written.**
3. **AI-use disclosure (allowed, but you must disclose).** You may use AI assistants as a learning and debugging aid. You **must** include a `TARGETS/AI-DISCLOSURE.md` file stating which tools you used and for what. The artifacts must be your team's own work on your own assigned tag and machine. Undisclosed or misrepresented AI use is an academic-integrity violation.

Submit a ZIP named `<student_id_1>_<student_id_2>…_<student_id_n>.zip` containing your `README.md`, your report, and a `TARGETS/` folder laid out as:

```
TARGETS/
├── secrets.csv          # single line: team,api_key_01,user_01,api_key_02,user_02,layer_sha256
├── AI-DISCLOSURE.md
├── target-2/            # recovered .env from insecurelabs01:<your tag>
├── target-4/            # extracted layer file (app/.env) from insecurelabs02:<your tag> + its sha256
├── target-12/           # your capsh --print and /proc/1/status Cap* dump
├── target-14/           # your privileged-container capsh --print / proc Cap* dump
└── target-15/           # host fdisk -l / lsblk + mount transcript, and the exploit video
```

The required artifact and the data-bound question for the artifact-bearing questions:

| Question | Required artifact (under `TARGETS/target-<n>/`) | Answer this (from *your* tag / host) |
| -------- | ----------------------------------------------- | ------------------------------------ |
| **Q2** | The recovered `.env` and the `docker run … cat .env` transcript for **your** `insecurelabs01` tag | What are the `API_KEY` and `USER` baked into your tag? |
| **Q4** | The extracted layer file (`app/.env`) from **your** `insecurelabs02` tag, plus the `sha256` of the layer it was added in | What is your recovered `API_KEY`, and the `sha256` of the layer that first added it? Why does your layer digest differ from another team's? |
| **Q12/Q14** | Your `capsh --print` output and `grep Cap /proc/1/status` dump, captured on your machine | Decode your `CapEff`: which capabilities are present, and how does the default set (Q12) differ from the privileged set (Q14)? |
| **Q15** | Your `fdisk -l`/`lsblk` output, the `mount` command transcript, and the exploit video | Which host device did you mount, what is its size, and what file did you read/create on the host to prove the compromise? |


> **AI-usage disclosure (this document).** Parts of this lab description, including the per-team anti-fabrication scaffolding and some wording, were drafted and reviewed with the help of AI assistants. All content was reviewed and validated by the teaching team, who remain responsible for it.

# Part 1: Keeping Secrets OUT of Docker Images

Researchers[^P12] have found that many publicly available Docker images contain secrets, such as private keys, API keys and more. Let’s start by spending about 10-15 minutes having a look at the research paper. You only need to get a brief overview of it.

## Setup

For this part of the Lab, you will need a computer with Docker installed. Other tools you might need (like a way to unpack tar files), should be available in most OSes.

## Start: Noob Mistake

Someone at Insecure LabsTM thought it was a good idea to ask Scott, the intern, to create the container for their latest product (a javascript application running on Node.js). He followed the [Docker getting started guide](https://docs.docker.com/get-started/workshop/02_our_app/) and adopted a similar Dockerfile:

```
FROM  node:18-alpine
WORKDIR /app  
COPY . .  
RUN yarn install  --production  
CMD  ["node", "src/index.js"]  EXPOSE 3000  
```

The application uses some credentials at runtime and creates an encrypted secure storage for them. However, during “yarn install”, it needs to have the following .env file (don’t question these design decisions, Tom’s a [super-genious](https://thedailywtf.com/articles/the-inner-json-effect?ref=gorillasun.de)!): 

```
API_KEY=<some  secret api key>  
USER=<some  secret user>  
```

This is how the folder of the application looks like when Scott builds the container:

```
├── insecure-labs/
│ ├── .env
│ ├── package.json
│ ├── README.md
│ ├── src/
│ └── yarn.lock
```

The image is publicly available in Docker Hub:

- https://hub.docker.com/r/isepdei/insecurelabs01

**Question** **1.** **What is the issue here?**

**Question** **2.** **Can you recover the actual secrets from the image? Describe how you would do this and include a screenshot of the recovered secrets.**

## Fixed it!

Tom, the senior engineer, quickly spotted Scott’s mistake: “Let me fix that for you!”. Here is the Dockerfile Tom wrote:

```
FROM  node:18-alpine  
WORKDIR /app  
COPY . .  
RUN yarn install  --production  
RUN rm .env # .env  only needed during yarn install
CMD  ["node", "src/index.js"]  EXPOSE 3000  
```

Scott created a new API key and invalidated the old one. The resulting image is publicly available in Docker Hub:

- https://hub.docker.com/r/isepdei/insecurelabs02

**Question** **3.** **What do you think of Tom’s fix?**

**Question** **4.** **Is it possible to recover the new API key now[^P13] ? Provide detailed instructions and include a screenshot of the recovered API key.**


**Question** **5.** **So, how would you go about doing Scott’s job? Would you do something different? Explain in detail.**

------

[^P11]: https://docs.docker.com/engine/security/

[^P12]: Markus Dahlmanns, Constantin Sander, Robin Decker, and Klaus Wehrle. 2023. Secrets Revealed in Container Images: An Internet-wide Study on Occurrence and Impact. In Proceedings of the 2023 ACM Asia Conference on Computer and Communications Security (ASIA CCS '23). Association for Computing Machinery, New York, NY, USA, 797–811. https://doi.org/10.1145/3579856.3590329. **Available in Moodle**.

[^P13]: Many sources on the Web to help you, but beware that very few are completely accurate; you will have to get it right by combining multiple sources. Here is a hint: you might want to take a **Dive**: https://github.com/wagoodman/dive **and** save the Docker image to a .tar file: https://docs.docker.com/reference/cli/docker/image/save/

