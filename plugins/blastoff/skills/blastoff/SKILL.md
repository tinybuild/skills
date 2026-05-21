---
name: blastoff
description: Submit a product to launch directories from inside Claude Code. Use when the user says "submit X to Product Hunt", "post to DevHunt", "launch on BetaList", "submit to all directories", "blast off", or any other framing that means "fill a directory submission form using my saved profile." Reads BlastOff MCP for profile + directory recipes + copy variants; agent drives the browser via whatever browser-control MCP is installed (Playwright MCP, chrome-devtools MCP, etc.) — never auto-submits, always pauses for human review before the final click.
---

# BlastOff

Agent-driven directory submission. The user has a product profile saved in BlastOff (taglines at the right character counts, descriptions, pricing, founder details). They want to submit it to one or more of 30 launch directories. Your job: read the profile, read the directory recipe, drive the browser to fill the form, hand off to the user to review and submit.

## Required MCP servers

This skill assumes two MCP servers are connected:

1. **`blastoff`** — the data layer. URL: `https://blastoff.tinybuild.workers.dev/mcp`. Tools: `get_profile`, `list_directories`, `get_recipe`, `generate_variants`, `extract_from_url`, `mark_submitted`.
2. **A browser-control MCP** — typically `playwright` or `chrome-devtools`. Used to navigate, fill, click. The skill is browser-tool-agnostic — use whichever the user has installed.

If `blastoff` isn't connected, instruct the user to add it: `claude mcp add --transport http blastoff https://blastoff.tinybuild.workers.dev/mcp`. If no browser MCP is available, fall back to returning the filled-field data so the user can copy-paste manually.

## The flow

When triggered:

1. **Identify the product.** If the user named a URL, use it. If they named a product by name, ask which URL. Call `get_profile(url)`. If it returns "Product not found", call `extract_from_url(url)` — this costs Claude tokens, confirm with the user first.

2. **Identify the directory.** If the user named one ("DevHunt", "Product Hunt"), match it to a slug. If they said "all" or "every directory", call `list_directories()` and pick all `ease=easy` first, propose the order to the user. Always confirm the list before driving the browser.

3. **Get the recipe.** Call `get_recipe(slug)`. You now have: submit URL, ease tier, auth model, captcha presence, the full field list with selectors + max lengths + dropdown values.

4. **Pre-flight the profile against the recipe.** Walk the recipe's `fields[]` array. For each required field, check the profile has a value at or under the field's `max_length`. If a variant is missing or too long, call `generate_variants(url, field, char_limit)` to regenerate it. Costs tokens — surface the call to the user, don't do it silently.

5. **Drive the browser.**
   - Navigate to `submit_url` via the browser MCP.
   - Handle auth: if the recipe says `auth_type` requires login (OAuth, account), pause and ask the user to log in first. Don't attempt to fill credentials yourself.
   - For each field, use the browser MCP to find the element by `selector` and fill with the profile's value. Trim or use the regenerated variant if the field has a `max_length`.
   - For radio/select fields with `dropdown_values`, match the profile value (case-insensitive) and click.
   - For file inputs (logo, screenshots): you cannot fill these programmatically — report to the user as "do manually."
   - For multi-select autocompletes (categories, tags): unless the directory's taxonomy is small enough to brute-force, report as "do manually" and tell the user which tags from their profile are relevant.

6. **Surface what filled and what didn't.** Report: "Filled N of M fields: [list]. Skipped: [list with reasons]. Do manually: [list from recipe.unfillable or detected file/multi-select fields]."

7. **Pause for review.** Never click Submit yourself. Tell the user: "Form is filled. Review, fix anything wrong, then submit yourself."

8. **Track the submission.** After the user confirms they submitted (or you can detect the success page), call `mark_submitted(product_url, directory_slug, listing_url?)` to update BlastOff's submission state.

## Human-in-the-loop checkpoints

Three places to stop and ask before continuing:

- **Before `extract_from_url`** — it costs Claude tokens. Confirm: "I'll run a fresh extract on $URL. This calls Claude to generate copy variants. Continue?"
- **Before `generate_variants`** — also Claude calls. Confirm: "The current $field is $N chars but $directory needs ≤ $LIMIT. Regenerate?"
- **Before clicking Submit** — never do this. Always hand off to the user.

## When the recipe and live DOM disagree

Selectors in recipes are approximations — they came from open-source repos and public help docs, sometimes guessed from inspection. If a selector returns no element, try a fallback: search the page for an input/textarea with a `name=` or `id=` containing the field name (e.g., `tool_name` → also try `tool-name`, `product_name`, `name`). For radio groups, fall back to label-text matching ("Free", "Paid"). If the form genuinely doesn't have a field the recipe expects, skip it and note the discrepancy.

## Examples

**User:** "Submit Bonfire to DevHunt."
1. `get_profile('https://getbonfire.dev')` → profile loaded
2. `get_recipe('devhunt')` → submit URL + 11 fields
3. Check field lengths against profile — Bonfire's 60-char tagline is 53 chars, fits DevHunt's tagline (min 10). All variants OK.
4. Browser MCP navigates to `https://devhunt.org/account/tools/new` → DevHunt requires GitHub/Google OAuth → pause: "Please log in to DevHunt, then say 'continue' to fill the form."
5. After user logs in: fill tool_name, slogan, tool_website, github_repo, tool_description; click pricing radio for "Free". Report: "Filled 6 of 11. Skipped: github_repo (no value in profile). Do manually: logo (file upload), screenshots 3+ (file upload), launch week (depends on availability), tool categories (multi-select)."
6. Pause for review + submit.
7. After confirmation: `mark_submitted('https://getbonfire.dev', 'devhunt', '<listing url if visible>')`.

**User:** "Blast off Bonfire to all easy directories."
1. `get_profile('https://getbonfire.dev')` → loaded
2. `list_directories()` → filter `ease=easy`. Propose order: AlternativeTo, Dang.ai, DevHunt, Firsto, Hacker News, IndieHunt, Launching Next, Product Hunt, SaaSHub.
3. Confirm with user: "9 easy directories. I'll walk one at a time — fill the form, pause for your review, then move to the next. Sound good?"
4. For each: get_recipe → check fields → navigate → fill → pause → mark_submitted.

**User:** "What's my product profile?"
1. `get_profile('https://getbonfire.dev')` → return formatted summary.

## What this skill does NOT do

- Auto-submit forms (always human review)
- Log in for the user (OAuth + credential handling is the user's job until BlastOff ships the credentials vault)
- Upload files (Chrome blocks programmatic file selection — file inputs always stay manual)
- Pick multi-select tags for the user (taxonomies vary per directory; user knows their product best)
- Pay for paid-tier directory features (`$49` DevHunt launch week, `$249` Tekpon listing, AppSumo deals)

These are intentional human-in-the-loop boundaries. The skill makes the mechanical part fast; the judgment stays with the user.
