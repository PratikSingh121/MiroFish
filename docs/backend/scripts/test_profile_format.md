# `backend/scripts/test_profile_format.py` — Profile Format Test Script

## Overview

This script is a standalone developer utility used to validate that generated OASIS profiles match the format expected by the downstream simulation scripts.

It checks both output formats used by MiroFish:
- Twitter CSV
- Reddit JSON

---

## `test_profile_formats()`

Creates two sample `OasisAgentProfile` instances, writes them into a temporary directory using the profile generator’s internal save helpers, and prints the results.

What it validates:
- Twitter profile CSV headers and required fields
- Reddit profile JSON structure and required fields
- presence of optional Reddit metadata such as profession and interested topics

This is useful when changing the profile generator or integrating with a new OASIS version.

---

## `show_expected_formats()`

Prints human-readable examples of the expected Twitter CSV and Reddit JSON formats.

This is a documentation-by-example helper for developers.

---

## Execution

When the script is run directly, it executes both:
1. `test_profile_formats()`
2. `show_expected_formats()`

---

## Related Files

- [`../app/services/oasis_profile_generator.py`](../app/services/oasis_profile_generator.md)
- [`run_twitter_simulation.py`](run_twitter_simulation.md)
- [`run_reddit_simulation.py`](run_reddit_simulation.md)
