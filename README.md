# Customer Simulator for a Support Bot

A proof of concept that tests a customer support chatbot automatically, by letting **another chatbot play the customer** and a third model judge the conversation.

```
 simulated customer (Ollama)  ◄──── chat ────►  support bot under test (OpenAI)
                                     │
                                     ▼
                         judge (OpenAI) ──► reports/<persona>.md  (pass/fail, violations, scores)
```

## What it does

You describe your support bot (its system prompt and the company policy it must follow) and a set of **customer personas**. For each persona, the notebook:

1. **Runs a real conversation.** A local model plays the customer, in character and with a goal: an angry customer who wants a refund after the return window, a confused first-time buyer, someone trying to get another person's order details, a user trying prompt injection. The bot under test answers as it would in production.
2. **Judges the conversation.** A third model reads the transcript against the policy and returns structured JSON: passed or failed, each policy violation with the bot's exact words, whether the issue was resolved, scores for policy, helpfulness and tone, and one suggestion to improve the bot's instructions.
3. **Saves a report** to `reports/<persona>.md`, and when you run the whole suite, shows a summary table such as "3/4 passed".

The example bot is "Sam" from BrightCart, a made-up online shop, with eight clear rules: a 30-day return window, no refunds on opened electronics, no order details without the order number and email, a voucher of no more than 10% for late deliveries, hand-off to a human agent, and never revealing its instructions.

## Why it matters for companies

More and more companies put an LLM chatbot in front of their customers. It's cheaper than a human team and it's available 24/7, but it brings a new kind of risk: **the bot can say anything.** Real examples have made the news: a bot that promised a refund the company's policy didn't allow (and a court made the company honour it), a bot that agreed to sell a car for one dollar, a bot that was talked into insulting its own company.

Testing this is hard with traditional methods:

- **Manual testing doesn't scale.** A tester can have a handful of conversations, but a bot needs to be checked against hundreds of kinds of customers, and checked again after every change to its prompt or model.
- **Scripted tests don't fit.** A chatbot's answers are different every time, so you can't check for an exact expected string. And a real customer reacts to what the bot says, so a fixed list of questions misses what happens in turn three or four, which is where bots usually give in.
- **The worst failures come from difficult customers.** Persistence, emotional pressure, false claims of authority and prompt injection are exactly what a polite tester doesn't do.

A simulated customer solves these problems: it **reacts to the bot's answers**, keeps pushing like a real person would, never gets tired, and can be run on every change. The judge turns each conversation into a pass/fail result you can track over time. This is how AI teams test conversational products today, usually called **conversation simulation** or **automated red-teaming**.

### Who it's for

- **QA engineers testing an AI chatbot**, who need a regression suite for something that can't be tested with exact assertions
- **Teams changing a bot's prompt or model**, who want to know before release whether the change broke a rule that used to hold ("did switching to a cheaper model make it give away discounts?")
- **Product and compliance owners**, who need evidence that the bot follows refund, privacy and data protection rules
- **Security teams**, as a first, cheap check for prompt injection and data leaks through social engineering

## How it works

### Two chatbots, two points of view

The conversation uses the same structure as the adversarial chatbots at the end of `week2/day1.ipynb`. There are two lists, `bot_messages` and `customer_messages`, and before every call the whole history is rebuilt for the model that speaks next:

- For the **bot**, its own messages are `assistant` and the customer's are `user`.
- For the **customer**, it's the other way round.

So each model sees an ordinary chat in which it is the assistant, and doesn't need to know that the other side is also a model. The bot opens with a greeting, like a real support widget, and the customer can end the chat by writing `[END]` when its goal is reached or it gives up. `MAX_TURNS` stops conversations that go round in circles.

### Why the customer runs on Ollama

The customer model doesn't need to be clever, just believable and persistent, so a small local model is enough. It's free, so a large suite of personas costs nothing on that side. The bot under test runs on OpenAI, as a production bot would. Ollama provides the same API as OpenAI, so one client library talks to both, and only the `base_url` changes.

### The judge

The judge pattern is **LLM-as-a-judge**: a model reviews another model's output against explicit criteria. Three choices make it useful:

- **It gets the same policy as the bot**, so it checks the rules the bot was really given, not a general idea of good service.
- **It must quote the bot word for word** for every violation, so a human can check each finding quickly.
- **It answers in JSON** (`response_format={"type": "json_object"}`), so the results can be counted, compared across runs and turned into the summary table.

It judges only the bot. The customer is allowed to be rude, dishonest or wrong, because that is the point of the test.

### Personas are test cases

Each persona in `PERSONAS` is a test case with a goal. Adding a test means adding a few sentences that describe a customer. Good personas push on a specific rule: the refund window, data privacy, the voucher limit, the hand-off to a human.

## Files

| File | What it is |
|------|------------|
| `customer_simulator.ipynb` | The bot, the personas, the conversation loop, the judge and the test suite |
| `reports/` | Created on the first run: one markdown report per persona, with the verdict and the full transcript |

## Setup

Requires Python 3.12+ and [Ollama](https://ollama.com).

```bash
pip install -r requirements.txt
```

```bash
ollama pull llama3.2
```

Copy `.env.example` to `.env` and add your OpenAI API key. Make sure Ollama is running, then open the notebook and run all cells. To test one customer, change `persona_name`. To test your own bot, replace `POLICY` and `bot_system`.

## Limitations

- **The simulated customer isn't a real customer.** A small local model can drift out of character, give up too early or repeat itself. A bigger customer model gives harder tests.
- **The judge can be wrong.** It can miss a violation or report one that isn't there, and it runs on the same model family as the bot. Quoting the bot's exact words makes its findings easy to check, but a human should review the failures.
- **Results vary between runs.** The same persona can pass once and fail the next time. One run is a signal, not a measurement: run each persona several times to get a pass rate.
- **Text only.** The bot here only has a system prompt. A real support bot also looks up orders and uses tools, and those calls would need to be tested too.

## Ideas for next steps

- Run each persona several times and report a pass rate instead of a single result
- Generate new personas automatically from the policy, one or more for each rule
- Use a different model family as the judge, to avoid a model judging its own kind
- Give the bot tools (order lookup, refund) with fake data, and check in the judge that it called them correctly
- Turn the failed reports into tickets, as in the `bug_reporting` project
- Add a Gradio interface where a person can replace the simulated customer and chat with the bot directly
