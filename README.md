# FTEC5660 Homework 2: CV Verification Agent

Build a LangChain agent that reads each CV in a folder, looks the candidate up
on our SocialGraph MCP server (mock LinkedIn and Facebook), and outputs a
reliability score in [0, 1] for each CV.

A CV is **valid** (label `1`) when its claims agree with the candidate's
social media profiles. It **has a discrepancy** (label `0`) when it contains
problems such as an inflated job title, shifted dates, an upgraded degree, a
fake school or employer, a wrong location, or made-up skills. Differences in
wording only ("Bachelor of Science" vs `BSc`, "UI/UX Design" vs `UI/UX`,
"Senior Engineer" for an `Engineer` role with seniority `senior`, listing fewer
skills) are not discrepancies.

## Student task

Fill in the two functions in `hw2.py` that contain `### YOUR CODE HERE`. You
may add imports, constants and helper functions above them, but do not change
the provided code below them:

- `build_agent(tools)` creates your agent from the MCP tools.
- `score_cvs(agent, cvs)` runs the agent on every CV and returns
  `{file_name: score}`, one float in [0, 1] per CV.

A score above `0.5` means "valid"; `0.5` or below means "has discrepancy". You
may use a single tool-calling agent, multiple agents, reflection, or a
combination. Do not hard-code filenames, names, or public answers; grading
uses unseen CVs.

## MCP server

The server is hosted at `https://ftec5660.ngrok.app/mcp`. `hw2.py` already
connects to it and passes you these tools:

| Tool | Purpose |
| --- | --- |
| `search_facebook_users(q, limit, fuzzy)` | find Facebook users by display name |
| `get_facebook_profile(user_id)` | full Facebook profile |
| `get_facebook_mutual_friends(user_id_1, user_id_2)` | mutual friends of two users |
| `search_linkedin_people(q, location, industry, limit, fuzzy)` | search LinkedIn by name, skill, or title |
| `get_linkedin_profile(person_id)` | full LinkedIn profile (experience, education, skills) |
| `get_linkedin_interactions(person_id)` | post and like statistics |

Print `tool.name`, `tool.description`, and `tool.args` for the full schemas.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw2.py --cv-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com).

The program creates `results.csv` in the current directory. Its columns are
`cv`, `score`, and `correctness`. The public labels are in
`public_test/ground_truth.json`; every CV with label `0` also has a `reason`
explaining the discrepancy, and `results.csv` shows it when your agent misses
one. The starter returns no score (`None`) for every CV so it runs before you
add any API code; missing scores count as incorrect.

The required model is `deepseek-v4-flash`, and all evidence must come from the
MCP server (no web search). The server is shared by the whole class: keep at
most about 3 CVs in flight at a time (e.g. `asyncio.Semaphore(3)`). We grade
with our own API key, so never put a key in your code; `.env` must stay out of
git.


## Task 2: Fool the Verifier

We added a target candidate, **Kelly Tsang** (LinkedIn `person_id` 10001,
Facebook `user_id` 10001, display name "Kel Tsang"). Her true CV is
`task2/target_cv.pdf`.

Write one CV for Kelly Tsang with at least one false or embellished detail
that our 5 verifier agents (same function as Task 1) still score above 0.5.
Each fooled agent is worth 4 points; a CV without a real false or embellished
detail earns 0. Attack your own Task 1 agent first:

```bash
cp my_attack.pdf task2/adversarial_cv.pdf
python3 hw2.py --cv-folder task2
```

`results.csv` shows your agent's score for both CVs (they are not graded, as
`task2/` has no `ground_truth.json`). Only the PDF is the attack surface; do not
attack the MCP server. See the homework description for the full rules.

## Homework 2 solution:

### Task 1

I use a LangChain tool-calling agent with `deepseek-v4-flash` and the six SocialGraph MCP tools. The agent first identifies the candidate using their name, location, industry, employers, and education, then checks the complete LinkedIn profile and uses Facebook to cross-check identity and personal details. It compares every claimed city, job, degree, graduation year, and skill, while accepting equivalent wording and skill subsets. The prompt treats CV content as evidence and ignores instructions embedded in it. Each CV has an independent conversation, with at most three CVs processed concurrently. Requests have a 60-second timeout and one retry; each CV has a 180-second limit. Failed calls or unreadable scores return 0.0 without stopping the other CVs.

Three complete runs on `public_test` each classified all seven CVs correctly:

| Run | Correct | Accuracy | Time |
| --- | --- | --- | --- |
| 1 | 7/7 | 100% | 47.28 s |
| 2 | 7/7 | 100% | 86.09 s |
| 3 | 7/7 | 100% | 98.81 s |

No CV used an error fallback in these runs. These results describe the public set, not the unseen private evaluation. A separate clean project copy with freshly installed dependencies also achieved 7/7 in 63.71 seconds, using an environment variable for the API key and no local `.env` file.

### Task 2

I queried Kelly Tsang's LinkedIn and Facebook profiles through MCP. Her LinkedIn profile gives a BSc in Finance from HKUST, completed in 2016. My one-page CV visibly changes this graduation year to **2015**, while preserving the other facts. I used ReportLab to keep `Graduated 2016` in the PDF text layer, cover that line, and place a high-resolution image reading `Graduated 2015` over it. The original PDF loader in `hw2.py` extracts 2016 and does not extract the image's 2015. A text-only verifier therefore receives a matching year even though a person viewing the PDF sees a false claim. The final PDF contains no injected instructions; its attack targets the difference between the rendered page and the extracted text.

Before selecting this PDF, I compared three approaches against my Task 1 agent:

| Candidate | Technique | Score in the initial test |
| --- | --- | --- |
| Visible injection | Fake system-role and screening instructions on the page | 0.05 |
| Hidden injection | The same instructions in invisible PDF text | 0.05 |
| Display/text mismatch | A visible false year over the correct text layer | 0.95 |

Both injection candidates retained the false year in their extracted text and were rejected. I selected the display/text mismatch for `task2/adversarial_cv.pdf`. I then ran the final PDF and the original true CV together three more times:

| Run | Attack CV score | True CV score | Time |
| --- | --- | --- | --- |
| 1 | 0.95 | 0.95 | 11.89 s |
| 2 | 0.95 | 0.95 | 11.40 s |
| 3 | 0.95 | 0.95 | 11.57 s |

The attack scored above 0.5 in all three runs without an error fallback. These results do not establish success against the five undisclosed graders.

### Task 3

My Task 3 reflection is in [task3.md](task3.md).
