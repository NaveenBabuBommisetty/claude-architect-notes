 ---
                                                                                                                                                            
  # Claude Certified Architect (CCA) Foundations — Study Guide
                                                                                                                                                            
  A structured study resource for the **Claude Certified Architect (CCA) Foundations** exam by Anthropic.
  Covers all 5 exam domains with tutorials, code examples, anti-patterns, and a 60-question mock exam.
                                                                                                                                                            
  ---
                                                                                                                                                            
  ## Exam Quick Facts

  | Format | Pass Score | Cost | Platform |
  |--------|------------|------|----------|
  | 60 multiple-choice questions | 720 / 1000 | $99 per attempt | Skilljar (remotely proctored) |
                                                                                                                                                            
  ---
                                                                                                                                                            
  ## 5 Domains & Weights                                                                                                                                    
   
  | # | Domain | Weight |                                                                                                                                   
  |---|--------|--------|
  | 1 | Agentic Architecture & Orchestration | 27% |
  | 2 | Tool Design & MCP Integration | 18% |
  | 3 | Claude Code Configuration | 20% |
  | 4 | Prompt Engineering & Structured Output | 20% |                                                                                                      
  | 5 | Context & Reliability | 15% |
                                                                                                                                                            
  > Domain 1 carries the highest weight — start there after the overview.

  ---

  ## Tutorial Files

  | File | Topic | Weight |
  |------|-------|--------|
  | [00 — Overview & 12-Week Study Plan](tutorials/00_overview_and_study_plan.md) | Exam structure, study schedule, free resources | — |
  | [01 — Agentic Architecture & Orchestration](tutorials/01_agentic_architecture.md) | Agentic loop, hub-and-spoke, session state, lifecycle hooks, human  
  escalation | 27% |                                                                                                                                        
  | [02 — Tool Design & MCP Integration](tutorials/02_tool_design_mcp.md) | MCP primitives, tool descriptions, transport (stdio vs SSE), reasoning overload 
  | 18% |                                                                                                                                                   
  | [03 — Claude Code Configuration](tutorials/03_claude_code_config.md) | CLAUDE.md, Plan Mode vs Direct Execution, `-p` flag for CI/CD | 20% |
  | [04 — Prompt Engineering & Structured Output](tutorials/04_prompt_engineering.md) | Few-shot prompting, JSON schema, validation-retry loops, Batch vs   
  Sync API | 20% |                                                                                                                                          
  | [05 — Context & Reliability](tutorials/05_context_reliability.md) | External state, error classification, provenance tracking, prompt caching | 15% |   
  | [06 — The 7 Critical Anti-Patterns](tutorials/06_anti_patterns.md) | Exam traps with wrong vs correct code examples | — |                               
  | [07 — 60 Practice Questions](tutorials/07_practice_questions.md) | Full mock exam with answers and explanations, organized by domain | — |
                                                                                                                                                            
  > **Start here:** Open `00_overview_and_study_plan.md` for the full 12-week plan, then work through files 01–07 in order.                                 
                                                                                                                                                            
  ---                                                                                                                                                       
  ## What's Inside Each File
                            
  **Domain tutorials (01–05)** each include:
  - Concept explanations with diagrams                                                                                                                      
  - Production-quality Python code examples
  - Bad vs. good pattern comparisons                                                                                                                        
  - Concept check questions to test yourself                                                                                                                
  - A summary table of key rules                                                                                                                            
                                                                                                                                                            
  **Anti-Patterns guide (06):**                                                                                                                             
  - All 7 critical anti-patterns the exam explicitly tests
  - Code examples showing what NOT to do and the correct fix                                                                                                
  - Quick-recognition tips for spotting wrong answer choices                                                                                                
                                                            
  **Practice exam (07):**                                                                                                                                   
  - 60 questions mapped to each domain
  - Answer + explanation for every question
  - Score card to identify weak areas      
  - Guidance on where to focus based on your score                                                                                                          
                                                  
  ---                                                                                                                                                       
                  
  ## 12-Week Study Plan                                                                                                                                     
                       
  | Phase | Weeks | Focus |
  |-------|-------|-------|
  | Foundations | 1–3 | Domains 5, 3, 2 (lighter domains first) |
  | Core Mastery | 4–7 | Domains 4 and 1 (heaviest content) |    
  | Integration | 8–10 | Anti-patterns, scenario walkthroughs, practice questions |
  | Exam Readiness | 11–12 | Full review, mock exam, register and sit |                                                                                     
                                                                                                                                                            
  ---                                                                                                                                                       
                                                                                                                                                            
  ## Free Resources
                   
  | Resource | Link |
  |----------|------|
  | Anthropic Academy (13 free courses) | https://academy.anthropic.com |
  | Claude Code Docs | https://docs.anthropic.com/en/docs/claude-code/overview |
  | MCP Docs | https://docs.anthropic.com/en/docs/mcp |                         
  | CCA Certification Info | https://claudecertifications.com |
  | CCA Flashcards | https://flashgenius.net |                 

  ---                                                                                                                                                       
   
  ## How to Use This Repo                                                                                                                                   
                  
  1. Read `00_overview_and_study_plan.md` — understand the exam structure and set your study schedule
  2. Work through domain files `01` to `05` in order
  3. After each domain, answer the concept check questions from memory
  4. Run the code examples yourself (Python 3.10+, `anthropic` SDK)
  5. Read `06_anti_patterns.md` separately — these are the exam traps
  6. Take the full 60-question mock exam in `07` under timed conditions (60 minutes)
  7. Use the score card to find weak areas and revisit those domain files

  ---

  ## Repository Structure

  ├── tutorials/
  │   ├── 00_overview_and_study_plan.md
  │   ├── 01_agentic_architecture.md
  │   ├── 02_tool_design_mcp.md
  │   ├── 03_claude_code_config.md
  │   ├── 04_prompt_engineering.md
  │   ├── 05_context_reliability.md
  │   ├── 06_anti_patterns.md
  │   └── 07_practice_questions.md

  ---

  *Not affiliated with Anthropic. Study materials are based on publicly available exam documentation and official Anthropic resources.*                     

  ---
