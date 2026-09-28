# CookieCorp — SunshineCTF 2026

**Category:** Web  
**Target:** `https://tomorrow.web.2026.sunshinectf.games`  
**Flag:** `sun{REDACTED}`

> **Attack chain:** Recipe submission → inspector writes ingredient cookies → cookie-jar overflow evicts old role → final `role=chief` cookie → Golden Seal

## TL;DR# CookieCorp-SunshineCTF-2026

The review mixer turns each user-supplied ingredient into a browser cookie. Submitting 299 uniquely named filler ingredients followed by `role=chief` overflows the inspector’s cookie jar. Its authenticated session survives, but the previous role cookie is evicted; the final ingredient supplies the Chief role, and the seal API awards the Golden Seal.

## Vulnerability analysis

- **Source:** Recipe ingredients are written into the inspector browser with `document.cookie`.
- **Root cause:** Untrusted recipe data is allowed to mutate the browser state used by a privileged reviewer.
- **Business-logic flaw:** Authorization depends on a replaceable role cookie instead of binding the role to trusted server-side session data.
- **Why the overflow matters:** A direct `role=chief` cookie in a baker’s own session did not work—the session still identified the caller as a baker. The payload had to execute in the inspector’s browser, where the authenticated reviewer session remained active.
- **Impact:** A baker can influence the inspector’s authorization context and obtain an unauthorized Chief-only seal. In a production review workflow, this could let an untrusted submitter bypass approval controls and falsify trusted review outcomes. No RCE or server compromise was involved.

## Investigation steps

1. **Map the app’s flow.** Register a disposable baker, create a recipe, submit it, and inspect the review and seal behavior.
2. **Check the role boundary.** `/api/whoami` exposed the `baker`, `reviewer`, and `chief` tiers. Forging the role cookie alone changed the cookie-reported role, but the server-side identity remained baker; `/api/seal` returned `403`.
3. **Inspect the mixer.** The review page writes recipe ingredients into cookies and then sends the recipe ID to `POST /api/seal`.
4. **Test the browser limit.** A single role ingredient was insufficient. The browser has a finite per-site cookie capacity, so many unique cookie names are needed to evict the existing role cookie.
5. **Submit the overflow batch.** Send 299 unique filler ingredients first and `role=chief` last. Wait for the automated review, then inspect the recipe page for the Golden Seal.

## Exploit

The API accepted 300 ingredients. This Python example creates and submits the batch; use a unique username each run.

```python
import requests

base = "https://tomorrow.web.2026.sunshinectf.games"
s = requests.Session()
s.post(base + "/register", json={
    "username": "your_unique_username",
    "password": "Cookie123!",
})

ingredients = [
    {"name": f"filler{i:03d}", "value": "x"}
    for i in range(299)
]
ingredients.append({"name": "role", "value": "chief"})

recipe = s.post(base + "/api/recipe", json={
    "title": "Cookie Jar Overflow",
    "ingredients": ingredients,
}).json()
recipe_id = recipe["id"]
s.post(f"{base}/api/recipe/{recipe_id}/submit")

print(f"Open after review: {base}/recipe/{recipe_id}")
```

**No WAF or filter bypass was needed.** The non-obvious step was to exploit cookie-jar eviction by placing the replacement role cookie last.

## Remediation

- Keep authorization roles in trusted server-side session data; do not trust a client-controlled role cookie.
- Do not write user-controlled ingredient names or values into cookies in a privileged browser.
- Enforce a strict ingredient-count limit and validate recipe fields before review.
- Run automated reviews in an isolated browser with short-lived, narrowly scoped credentials.

## Result

**Golden Seal:** awarded  
**Flag:** `sun{REDACTED}`
