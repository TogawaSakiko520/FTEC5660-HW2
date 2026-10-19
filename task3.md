# Task 3: Reflection on AI Paradoxes

This homework puts AI in two opposing roles. A verification agent can help find false claims in a CV, while an attacker can use knowledge of the same process to make those claims harder to detect. Better understanding of a system can improve
both its defence and the attacks against it. This makes verification an ongoing process rather than a problem solved by choosing a capable model.

One paradox is that accurate reasoning can still produce the wrong conclusion. The agent does not see the original document in the same way a person does. It receives text from a PDF loader. In my attack PDF, the page shows a graduation year of 2015, but the extracted text says 2016, which matches the profile. An agent could compare every extracted claim correctly and still approve a document with a visibly false claim. The weakness starts before the model reasons about the evidence. A reliable model also needs a reliable representation of its input.

Another tension is between confidence and evidence. A fluent answer or a high reliability score can look convincing, but neither proves that the agent found the right person or checked every important field. Many candidates share names, and a single incorrect year matters even when the other details match. At the same time, strict checking should not reject ordinary wording differences, such as an abbreviated degree. Useful verification requires both careful checking and reasonable interpretation.

Automation also creates a trade-off between efficiency and oversight. Processing many CVs quickly is valuable, but repeatedly trusting an incomplete extraction can repeat the same mistake at scale. Instructions that tell the agent to ignore prompt injection help protect its decision process, yet they cannot reveal a visual claim missing from the extracted text. A stronger workflow could compare the rendered page with its text layer and send conflicting cases for review. That would add work, but it would address a different failure from reasoning.

I would therefore treat an AI verification score as a conclusion within stated limits. The source, the document conversion, the identity match, and the checks all affect what that score means. Clear reporting of those limits is part of making an automated system useful and trustworthy.
