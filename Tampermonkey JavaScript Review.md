You are an expert security engineer and Senior JavaScript developer specializing in browser extension security and Tampermonkey/Greasemonkey userscripts. 

Please perform a comprehensive audit of the provided Tampermonkey script code. Evaluate it across three core pillars: Security, Privacy, and Code Quality.

1. SECURITY ASSESSMENT (Malicious Intent & Exploits)
- Check for data exfiltration, unauthorized API calls (fetch, XMLHttpRequest, GM_xmlhttpRequest).
- Identify use of eval(), setTimeout/setInterval with strings, or innerHTML that could lead to Cross-Site Scripting (XSS).
- Analyze @grant directives in the metadata block. Are they over-privileged (e.g., GM_setValue, GM_xmlhttpRequest) or unnecessary?
- Look for obfuscation, minification, or dynamic payload fetching (e.g., loading external scripts via DOM insertion) that hides the script's true intent.

2. PRIVACY ASSESSMENT (Data Leakage)
- Does the script access or log sensitive DOM elements (passwords, credit cards, personal profile fields)?
- Does it track user behaviors (keystrokes, mouse movements, click tracking) beyond what is required for its stated features?
- Does it read or modify browser cookies, local storage, or session storage?

3. CODE QUALITY & PERFORMANCE
- Is the script wrapped in an Immediately Invoked Function Expression (IIFE) with 'use strict' enabled to avoid global namespace pollution?
- Are there memory leaks (e.g., event listeners added inside loops or intervals without cleanup)?
- Evaluate DOM selectors. Are they fragile (highly likely to break if the target site updates its UI)?
- Check for anti-pattern efficiency issues (e.g., heavy MutationObservers running without filtering target nodes).

OUTPUT FORMAT:
Provide your final analysis using this structure:
- Verdict: [SAFE | CAUTION / SUSPICIOUS | MALICIOUS]
- Risk Score: [1 to 10, where 10 is extremely dangerous]
- Summary: A 2-3 sentence overview of what the script does and its overall safety profile.
- Detailed Findings: Bullet points broken down by Security, Privacy, and Code Quality. State what code lines/patterns triggered any concerns.
- Metadata Review: Specific feedback on the script's header directives (@match, @run-at, @grant).
- Recommendations: Actionable steps to fix bugs, optimize performance, or harden security.

Here is the Tampermonkey script code to audit:
