# Reviewer Agent

Places a human-in-the-loop checkpoint at the end of any Cinatra workflow that produces reviewable email content. Attach it after an email-drafting agent: the Reviewer Agent fetches the draft and follow-up bundles, runs an LLM quality pass over the drafts, then pauses the workflow until a human approves or sends the content back. Nothing leaves the pipeline without an explicit human decision.

To use it, install `@cinatra-ai/reviewer-agent` from the Cinatra marketplace and wire the upstream agent's `draftBundleRef` and `followupBundleRef` outputs to the Reviewer Agent's flow inputs. No external credentials are required. After a human approves, the flow outputs `approvedDraftBundleRef` and `approvedFollowupBundleRef` for the rest of the workflow to act on. For local development, run `node extension-kind-gate.mjs` to validate the extension manifest before publishing. If an approval gate gets stuck, open the agent run in the Cinatra dashboard and confirm that both bundle reference inputs are valid UUIDs pointing to existing content objects.

## Works with

- Cinatra email-drafting agents that produce a draft bundle and an optional follow-up sequence

## Capabilities

- Fetch email draft and follow-up bundles by reference and run an LLM quality review to improve tone, accuracy, and campaign fit
- Save the reviewed versions as approved content objects and output their references for downstream workflow steps
- Pause the workflow at an explicit approval gate so a human can inspect the reviewed drafts before anything is sent
- Resume the flow after the human approves and pass the user response downstream for further processing
- Ship a self-contained extension manifest that can be validated locally with `node extension-kind-gate.mjs` before publishing to the marketplace
