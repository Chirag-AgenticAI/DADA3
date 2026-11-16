# huskyutils: the open issues

Seven fixes were verified in one verifier round each and are ready to review: #1 (slug double hyphens and trailing hyphen), #2 (shared payment losing a cent), #4 (delivery date skipping the weekend), #5 (overlapping delivery windows after merging), #6 (1 kg package charged $9 shipping), #8 (median wrong for an even number of days), and #10 (product card description overflowing). No issue is marked as needing a person. Three issues were closed without a code change: #3 (word_count and "don't stop"), #7 (free shipping over $50, an enhancement), and #9 (5 kg package in the heavy tier).

| Issue | Triage | What happened |
|---|---|---|
| #1 Slugs have double hyphens and a hyphen at the end | bug | fix verified, 1 verifier round(s) |
| #2 Shared payment loses a cent | bug | fix verified, 1 verifier round(s) |
| #3 word_count says "don't stop" is 2 words | not_a_bug | closed, no code change |
| #4 Delivery date skips over the weekend wrong | bug | fix verified, 1 verifier round(s) |
| #5 Delivery windows overlap after merging | bug | fix verified, 1 verifier round(s) |
| #6 A 1 kg package is charged $9 shipping | bug | fix verified, 1 verifier round(s) |
| #7 Free shipping for orders over $50 | enhancement | closed, no code change |
| #8 Median on the sales dashboard is wrong for an even number of days | bug | fix verified, 1 verifier round(s) |
| #9 5 kg package should be in the heavy tier | not_a_bug | closed, no code change |
| #10 Product card description runs past the edge of the card | bug | fix verified, 1 verifier round(s) |

## #1: Slugs have double hyphens and a hyphen at the end

**Triage:** bug. slugify replaces each non-alnum character with a hyphen individually, without collapsing runs or stripping leading/trailing hyphens. The docstring promises single hyphens per run and no leading/trailing hyphen, so the code fails its spec.

**Fix** (fix verified): slugify replaced each non-alnum char with its own hyphen, so runs produced double hyphens and trailing hyphens. I now map every non-alphanumeric character to a space and join the split words with single hyphens, which collapses runs and trims the ends. I also added accent stripping via NFKD and ASCII encoding, since the docstring requires é→e and the old code kept accents.

```diff
--- a/huskyutils/text.py
+++ b/huskyutils/text.py
@@ -1,4 +1,6 @@
 """Text helpers for product pages."""
+
+import unicodedata
 
 
 def slugify(title):
@@ -13,10 +15,12 @@
         slugify("USB-C Dock (2nd gen)") == "usb-c-dock-2nd-gen"
         slugify("Crème Brûlée Mug") == "creme-brulee-mug"
     """
-    out = []
-    for ch in title.lower():
-        out.append(ch if ch.isalnum() else "-")
-    return "".join(out)
+    # Decompose accents and drop non-ASCII bytes so "é" becomes "e".
+    ascii_text = unicodedata.normalize("NFKD", title.lower()).encode("ascii", "ignore").decode("ascii")
+    # Replace every non-alphanumeric character with a space, then split on runs of
+    # separators. This collapses repeated hyphens and drops leading/trailing ones.
+    cleaned = "".join(ch if (ch.isascii() and ch.isalnum()) else " " for ch in ascii_text)
+    return "-".join(cleaned.split())
 
 
 def word_count(text):
```

**Verifier:** Wrote tests/test_issue.py (9 tests: the two issue examples, the three docstring examples, lowercase ASCII, no leading/trailing or double hyphens, empty result for no alphanumerics, digits kept) in both folders.
original/: `python -m unittest discover -s tests -v` -> Ran 33 tests, FAILED (failures=8). The issue tests fail: 'webcam--hd-' != 'webcam-hd', 'hello--world--t-shirt' != 'hello-world-t-shirt'. Other failures are from my extra checks, which also fail on the original.
fixed/: `python -m unittest discover -s tests -v` -> Ran 33 tests, OK. All 9 new tests and the existing tests pass.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.text import slugify


class TestSlugifyIssue1(unittest.TestCase):
    def test_issue_webcam(self):
        self.assertEqual(slugify("Webcam (HD)"), "webcam-hd")

    def test_issue_hello_world(self):
        self.assertEqual(slugify("Hello, World! T-shirt"), "hello-world-t-shirt")

    def test_docstring_examples(self):
        self.assertEqual(slugify("Wireless Headphones"), "wireless-headphones")
        self.assertEqual(slugify("USB-C Dock (2nd gen)"), "usb-c-dock-2nd-gen")
        self.assertEqual(slugify("Crème Brûlée Mug"), "creme-brulee-mug")

    def test_lowercase_ascii(self):
        result = slugify("Crème Brûlée Mug")
        self.assertEqual(result, result.lower())
        self.assertTrue(all(ord(c) < 128 for c in result))

    def test_no_leading_or_trailing_hyphen(self):
        self.assertEqual(slugify("  (Deluxe) Kit!! "), "deluxe-kit")

    def test_no_double_hyphen(self):
        self.assertNotIn("--", slugify("a -- b"))
        self.assertEqual(slugify("a -- b"), "a-b")

    def test_no_letters_or_digits_gives_empty(self):
        self.assertEqual(slugify("!!! ---"), "")
        self.assertEqual(slugify("   "), "")

    def test_digits_kept(self):
        self.assertEqual(slugify("Model 3000 (v2)"), "model-3000-v2")


if __name__ == "__main__":
    unittest.main()
```

</details>

## #2: Shared payment loses a cent

**Triage:** bug. Docstring says shares sum to exactly cents, with first shares getting the extra cent; split_amount(1000,3) returns [333,333,333] summing to 999. The code ignores the remainder, so this is a bug.

**Fix** (fix verified): split_amount dropped the remainder (1000 into 3 gave [333,333,333]) and didn't handle negative amounts as the docstring specifies. It now splits the absolute value with divmod, gives the first `rem` shares the extra cent, and reapplies the sign, so 1000/3 gives [334,333,333] and -1000/3 gives [-334,-333,-333]. I followed the docstring (extra cent to the first shares), not the issue's suggestion of the last person.

```diff
--- a/huskyutils/money.py
+++ b/huskyutils/money.py
@@ -12,5 +12,6 @@
         split_amount(1000, 3) == [334, 333, 333]
         split_amount(-1000, 3) == [-334, -333, -333]
     """
-    share = cents // people
-    return [share] * people
+    sign = -1 if cents < 0 else 1
+    base, rem = divmod(abs(cents), people)
+    return [sign * (base + 1) if i < rem else sign * base for i in range(people)]
```

**Verifier:** Wrote tests/test_issue.py in both folders from the issue and the split_amount docstring (issue example sum, all three docstring examples including the negative case, and an exhaustive check that shares sum to the input and differ by at most one cent for cents in -50..50 and people in 1..7).
- original/: `python -m unittest discover -s tests -v` -> 30 tests, 4 failures in test_issue (sum 999 != 1000; [333,333,333] != [334,333,333]; [-334,-334,-334] != [-334,-333,-333]; sums off for negatives). Reproduces the issue.
- fixed/: same command -> Ran 30 tests, OK (exit 0). All existing tests and my tests pass.
Note: the issue suggests giving the extra cent to the last person, but the docstring says the first shares get it. The fix follows the docstring ([334, 333, 333]), which I treated as the spec. The issue text's suggestion conflicts with the spec; the fix's behavior matches the docstring.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.money import split_amount


class TestIssue2(unittest.TestCase):
    def test_issue_example_sums_to_full_amount(self):
        shares = split_amount(1000, 3)
        self.assertEqual(sum(shares), 1000)

    def test_docstring_uneven_split_first_shares_get_extra(self):
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])

    def test_docstring_even_split(self):
        self.assertEqual(split_amount(900, 3), [300, 300, 300])

    def test_docstring_negative_split(self):
        self.assertEqual(split_amount(-1000, 3), [-334, -333, -333])

    def test_shares_differ_by_at_most_one_and_sum(self):
        for cents in range(-50, 51):
            for people in range(1, 8):
                shares = split_amount(cents, people)
                self.assertEqual(len(shares), people)
                self.assertEqual(sum(shares), cents)
                self.assertLessEqual(max(shares) - min(shares), 1)
```

</details>

## #3: word_count says "don't stop" is 2 words

**Triage:** not_a_bug. word_count splits on whitespace, and the docstring explicitly says "don't" is one word. "don't stop" gives 2 words, matching the spec. The reporter wants contractions expanded, which the docstring does not promise.

## #4: Delivery date skips over the weekend wrong

**Triage:** bug. add_business_days adds calendar days first (start + days), then steps forward past weekends/holidays once. For Thu 11-05 + 3 days = Sun 11-08, which skips forward to Mon 11-09. The docstring requires counting business days, so 3 business days from Thursday should be Tuesday 11-10. The code does not count business days; it's a bug.

**Fix** (fix verified): add_business_days added calendar days and then skipped forward only once, so it did not count business days (Thu 11-05 + 3 gave Mon 11-09 instead of Tue 11-10). It now steps one day at a time and decrements a counter only on business days (weekday < 5 and not a holiday), which matches the docstring; days == 0 still returns start unchanged.

```diff
--- a/huskyutils/dates.py
+++ b/huskyutils/dates.py
@@ -15,7 +15,10 @@
     """
     if days == 0:
         return start
-    result = start + timedelta(days=days)
-    while result.weekday() >= 5 or result in holidays:
+    result = start
+    remaining = days
+    while remaining > 0:
         result += timedelta(days=1)
+        if result.weekday() < 5 and result not in holidays:
+            remaining -= 1
     return result
```

**Verifier:** Wrote tests/test_issue.py (7 tests: the issue's Thu 11-05 +3 -> Tue 11-10 example, all docstring examples, days=0, weekend start, result-is-business-day, holiday mid-span) and copied it into both folders.
original/: `python -m unittest discover -s tests -v` -> Ran 32, FAILED (failures=2). test_issue_example_thursday_plus_3 got 2026-11-09 instead of 2026-11-10; test_holiday_in_middle_of_span got 2026-11-11 instead of 2026-11-12. The original bug reproduces.
fixed/: same command -> Ran 32 tests, OK. All existing tests and all 7 new tests pass.

<details><summary>The verifier's test</summary>

```python
import unittest
from datetime import date

from huskyutils.dates import add_business_days


class TestIssue4(unittest.TestCase):
    def test_issue_example_thursday_plus_3(self):
        # Thu 2026-11-05 + 3 business days: Fri, Mon, Tue -> Tue 2026-11-10
        self.assertEqual(add_business_days(date(2026, 11, 5), 3), date(2026, 11, 10))

    def test_docstring_friday_plus_1(self):
        self.assertEqual(add_business_days(date(2026, 11, 6), 1), date(2026, 11, 9))

    def test_docstring_friday_plus_1_with_holiday(self):
        self.assertEqual(
            add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}),
            date(2026, 11, 10),
        )

    def test_days_zero_returns_start(self):
        self.assertEqual(add_business_days(date(2026, 11, 7), 0), date(2026, 11, 7))

    def test_weekend_start_plus_1_is_monday(self):
        # Sat 2026-11-07 + 1 -> Mon 2026-11-09
        self.assertEqual(add_business_days(date(2026, 11, 7), 1), date(2026, 11, 9))

    def test_result_is_business_day(self):
        start = date(2026, 11, 1)
        for n in range(1, 15):
            r = add_business_days(start, n)
            self.assertLess(r.weekday(), 5, f"n={n} gave {r}")
            self.assertGreater(r, start)

    def test_holiday_in_middle_of_span(self):
        # Mon 2026-11-09 + 2 with Tue 11-10 a holiday: Wed 11-11, Thu 11-12
        self.assertEqual(
            add_business_days(date(2026, 11, 9), 2, holidays={date(2026, 11, 10)}),
            date(2026, 11, 12),
        )


if __name__ == "__main__":
    unittest.main()
```

</details>

## #5: Delivery windows overlap after merging

**Triage:** bug. merge_intervals sorts nothing: it iterates in input order, so unsorted input yields [[8,10],[1,6]]. The docstring says input may be in any order and the result is sorted by start time. The code does not sort, so this is a bug.

**Fix** (fix verified): merge_intervals iterated in input order, so unsorted windows were merged incorrectly (e.g. [[8,10],[1,3],[2,6]] gave [[8,10],[1,6]]). I now iterate over sorted(intervals, key=start), which returns a new list and leaves the input untouched, as the docstring requires.

```diff
--- a/huskyutils/intervals.py
+++ b/huskyutils/intervals.py
@@ -14,7 +14,7 @@
         merge_intervals([[1, 4], [4, 5]]) == [[1, 5]]
     """
     merged = []
-    for start, end in intervals:
+    for start, end in sorted(intervals, key=lambda pair: pair[0]):
         if merged and start <= merged[-1][1]:
             merged[-1][1] = max(merged[-1][1], end)
         else:
```

**Verifier:** Wrote tests/test_issue.py (8 tests: the issue example, both docstring examples, unsorted overlap, empty, single, input not modified, returned list is new) into both folders. Ran `python -m unittest discover -s tests -v` in each.
original/: Ran 33 tests, FAILED (failures=3). The failures are in my issue tests: test_issue_example_unsorted returns [[8, 10]] instead of [[1, 6], [8, 10]], test_unsorted_overlap_merges returns [[5, 7]] instead of [[1, 7]], and test_input_not_modified fails for the same reason. So the tests reproduce the bug.
fixed/: Ran 33 tests, OK. All 8 of my tests pass, along with the existing test_intervals, test_cards, test_dates, test_money, test_shipping, test_stats and test_text tests.
The fix changes the loop to `sorted(intervals, key=lambda pair: pair[0])`. This keeps the docstring's behaviour: touching windows merge, the input is not modified, and the result is a new list.

<details><summary>The verifier's test</summary>

```python
import copy
import unittest

from huskyutils.intervals import merge_intervals


class TestIssue5(unittest.TestCase):
    def test_issue_example_unsorted(self):
        self.assertEqual(merge_intervals([[8, 10], [1, 3], [2, 6]]), [[1, 6], [8, 10]])

    def test_docstring_example_1(self):
        self.assertEqual(merge_intervals([[1, 3], [2, 6], [8, 10]]), [[1, 6], [8, 10]])

    def test_docstring_example_2_touching(self):
        self.assertEqual(merge_intervals([[1, 4], [4, 5]]), [[1, 5]])

    def test_unsorted_overlap_merges(self):
        self.assertEqual(merge_intervals([[5, 7], [1, 3], [2, 6]]), [[1, 7]])

    def test_empty(self):
        self.assertEqual(merge_intervals([]), [])

    def test_single(self):
        self.assertEqual(merge_intervals([[3, 4]]), [[3, 4]])

    def test_input_not_modified(self):
        data = [[8, 10], [1, 3], [2, 6]]
        snapshot = copy.deepcopy(data)
        result = merge_intervals(data)
        self.assertEqual(data, snapshot)
        self.assertEqual(result, [[1, 6], [8, 10]])

    def test_returns_new_list(self):
        data = [[1, 2], [3, 4]]
        result = merge_intervals(data)
        result[0][1] = 99
        self.assertEqual(data, [[1, 2], [3, 4]])


if __name__ == "__main__":
    unittest.main()
```

</details>

## #6: A 1 kg package is charged $9 shipping

**Triage:** bug. Docstring says up to and including 1 kg costs 500. Code uses weight_kg < 1 for 500, so weight 1 falls to the next branch and returns 900. Code contradicts docstring: bug.

**Fix** (fix verified): shipping_cost used `weight_kg < 1` for the $5.00 tier, so a package of exactly 1 kg fell through to the $9.00 tier. Changed the comparison to `weight_kg <= 1` to match the docstring's "up to and including 1 kg". No other lines changed.

```diff
--- a/huskyutils/shipping.py
+++ b/huskyutils/shipping.py
@@ -12,7 +12,7 @@
         shipping_cost(0.4) == 500
         shipping_cost(5.2) == 1100
     """
-    if weight_kg < 1:
+    if weight_kg <= 1:
         return 500
     if weight_kg <= 5:
         return 900
```

**Verifier:** Wrote tests/test_issue.py in both folders (10 tests: docstring examples shipping_cost(0.4)==500 and shipping_cost(5.2)==1100, plus boundary tests at 1, 1.0, 0.999, 1.001, 1.5, 5, 6, 6.01).
original/: `python -m unittest discover -s tests -v` -> 35 tests, FAILED (failures=2): test_exactly_1kg_is_500 (900 != 500) and test_exactly_1kg_float (900 != 500). The issue reproduces.
fixed/: same command -> 35 tests, OK, exit 0. All pre-existing tests and my 10 pass.
The fix changes `weight_kg < 1` to `weight_kg <= 1` in shipping_cost; the other boundaries (5 kg -> 900, 6 kg -> 1100) are unchanged and match the docstring.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.shipping import shipping_cost


class TestIssue6(unittest.TestCase):
    def test_docstring_example_light(self):
        self.assertEqual(shipping_cost(0.4), 500)

    def test_docstring_example_over_5(self):
        self.assertEqual(shipping_cost(5.2), 1100)

    def test_exactly_1kg_is_500(self):
        # Issue #6: up to and including 1 kg ships for $5.00
        self.assertEqual(shipping_cost(1), 500)

    def test_exactly_1kg_float(self):
        self.assertEqual(shipping_cost(1.0), 500)

    def test_just_below_1kg(self):
        self.assertEqual(shipping_cost(0.999), 500)

    def test_just_over_1kg_is_900(self):
        self.assertEqual(shipping_cost(1.001), 900)

    def test_1_5kg_is_900(self):
        self.assertEqual(shipping_cost(1.5), 900)

    def test_exactly_5kg_is_900(self):
        self.assertEqual(shipping_cost(5), 900)

    def test_6kg_is_1100(self):
        self.assertEqual(shipping_cost(6), 1100)

    def test_6_01kg_is_1300(self):
        self.assertEqual(shipping_cost(6.01), 1300)


if __name__ == "__main__":
    unittest.main()
```

</details>

## #7: Free shipping for orders over $50

**Triage:** enhancement. The issue asks for a new feature: free shipping over $50. shipping_cost's docstring only covers weight-based pricing and makes no promise about order totals, so this is a request for something the docstring does not promise.

## #8: Median on the sales dashboard is wrong for an even number of days

**Triage:** bug. median returns ordered[len//2] with no averaging for even counts. Docstring says even count gives average of two middle values, and median([4,1,3,2]) == 2.5. For [10,20,30,40] code returns 30, contrary to spec. Bug.

**Fix** (fix verified): median() returned only the upper middle element for even-length inputs. It now averages the two middle values when the count is even (odd counts are unchanged), so median([10,20,30,40]) == 25 and median([4,1,3,2]) == 2.5.

```diff
--- a/huskyutils/stats.py
+++ b/huskyutils/stats.py
@@ -11,4 +11,7 @@
         median([4, 1, 3, 2]) == 2.5
     """
     ordered = sorted(values)
-    return ordered[len(ordered) // 2]
+    mid = len(ordered) // 2
+    if len(ordered) % 2 == 1:
+        return ordered[mid]
+    return (ordered[mid - 1] + ordered[mid]) / 2
```

**Verifier:** Wrote tests/test_issue.py in both folders from issue #8 and the median docstring (issue example [10,20,30,40]==25, docstring examples [3,1,2]==2 and [4,1,3,2]==2.5, two values, single value, unsorted odd input, input not modified).
original/: `python -m unittest discover -s tests -v` ran 32 tests, 3 failed in test_issue (test_issue_example_even_count: 30 != 25; test_docstring_even_example: 3 != 2.5; test_two_values_average: 4 != 2.5). So the tests reproduce the bug.
fixed/: same command ran 32 tests, all OK (exit code 0), including all pre-existing tests.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.stats import median


class TestIssue8Median(unittest.TestCase):
    def test_issue_example_even_count(self):
        # Issue #8: median of 10, 20, 30, 40 must be 25, not 30.
        self.assertEqual(median([10, 20, 30, 40]), 25)

    def test_docstring_even_example(self):
        self.assertEqual(median([4, 1, 3, 2]), 2.5)

    def test_docstring_odd_example(self):
        self.assertEqual(median([3, 1, 2]), 2)

    def test_two_values_average(self):
        self.assertEqual(median([1, 4]), 2.5)

    def test_single_value(self):
        self.assertEqual(median([7]), 7)

    def test_odd_count_unsorted_input(self):
        self.assertEqual(median([9, 1, 5, 7, 3]), 5)

    def test_input_not_modified(self):
        values = [40, 10, 30, 20]
        median(values)
        self.assertEqual(values, [40, 10, 30, 20])


if __name__ == "__main__":
    unittest.main()
```

</details>

## #9: 5 kg package should be in the heavy tier

**Triage:** not_a_bug. Docstring says over 5 kg costs $9 plus $2 per started kg; up to and including 5 kg is $9.00. shipping_cost(5) returns 900, matching the docstring. The reporter wants a different tier boundary, which the docstring does not promise otherwise. Not a bug.

## #10: Product card description runs past the edge of the card

**Triage:** bug. The docstring says the result is at most limit characters including "...", and words are never cut. The code returns text[:limit] + "..." which is limit+3 characters (33 here) and cuts mid-word. This contradicts the spec, so it is a bug.

**Fix** (fix verified): truncate returned text[:limit] + "..." (limit+3 chars, cutting mid-word). I changed it to keep whole words from the start while the result plus "..." stays within limit, and fall back to text[:limit-3] + "..." only when the first word doesn't fit.

```diff
--- a/huskyutils/cards.py
+++ b/huskyutils/cards.py
@@ -15,4 +15,15 @@
     """
     if len(text) <= limit:
         return text
-    return text[:limit] + "..."
+    budget = limit - 3
+    kept = []
+    used = 0
+    for word in text.split():
+        needed = len(word) if not kept else used + 1 + len(word)
+        if needed > budget:
+            break
+        kept.append(word)
+        used = needed
+    if not kept:
+        return text[:budget] + "..."
+    return " ".join(kept) + "..."
```

**Verifier:** Wrote tests/test_issue.py in both folders from the issue and the docstring: the two docstring examples, the headphones case (len <= 30, ends with "..."), the first-word-too-long fallback, a length-never-exceeds-limit sweep for limits 10-44, whole-word/no-space-before-"..." checks where the first word fits, a maximal-words check, and the exact-length case.
original/: `python -m unittest discover -s tests` -> Ran 33 tests, FAILED (failures=194). The issue tests fail there (test_docstring_long_example, test_issue_headphones_fits_limit, test_first_word_too_long, test_keeps_as_many_words_as_fit, test_result_never_exceeds_limit, test_words_not_cut_and_no_space_before_ellipsis), so the test reproduces the issue.
fixed/: `python -m unittest discover -s tests -v` -> Ran 33 tests, OK. All existing tests and all of my tests pass.
Note: my first draft had a bug (it checked the whole-word property in the fallback case, where the docstring allows a mid-word cut). I corrected it to skip that case, and re-ran both folders after confirming the fixed/ copy matched the corrected file (identical md5).

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.cards import truncate


class TestIssue10(unittest.TestCase):
    # Docstring examples
    def test_docstring_short_unchanged(self):
        self.assertEqual(truncate("Wireless headphones", 30), "Wireless headphones")

    def test_docstring_long_example(self):
        self.assertEqual(
            truncate("Wireless noise-cancelling headphones with case", 30),
            "Wireless noise-cancelling...",
        )

    # Issue example: result must fit in 30 characters
    def test_issue_headphones_fits_limit(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertLessEqual(len(s), 30)
        self.assertTrue(s.endswith("..."))

    # Spec: if not even the first word fits, first limit-3 chars + "..."
    def test_first_word_too_long(self):
        self.assertEqual(truncate("Supercalifragilisticexpialidocious area", 10), "Superca...")
        self.assertEqual(len(truncate("Supercalifragilisticexpialidocious area", 10)), 10)

    # Spec: at most limit chars, for a range of inputs and limits (limit >= 10)
    def test_result_never_exceeds_limit(self):
        texts = [
            "Wireless noise-cancelling headphones with case",
            "Mechanical keyboard with brown switches",
            "USB-C dock with two displays and power delivery",
            "a b c d e f g h i j k l m n o p q r s t u v w x y z",
            "Supercalifragilisticexpialidocious area",
        ]
        for text in texts:
            for limit in range(10, 45):
                with self.subTest(text=text, limit=limit):
                    s = truncate(text, limit)
                    self.assertLessEqual(len(s), limit)

    # Spec: when the first word fits, result ends with "..." on whole words, no space before "..."
    def test_words_not_cut_and_no_space_before_ellipsis(self):
        text = "Mechanical keyboard with brown switches and rgb"
        for limit in range(10, 45):
            with self.subTest(limit=limit):
                s = truncate(text, limit)
                if s == text:
                    continue
                first_word = text.split()[0]
                if len(first_word) > limit - 3:
                    continue  # fallback case: mid-word cut is allowed by the spec
                self.assertTrue(s.endswith("..."))
                head = s[:-3]
                self.assertFalse(head.endswith(" "))
                self.assertTrue(text.startswith(head))
                self.assertTrue(text[len(head):].startswith(" "))

    # Spec: keeps as many whole words as fit (maximal)
    def test_keeps_as_many_words_as_fit(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertEqual(s, "Wireless noise-cancelling...")
        s2 = truncate("one two three four five six seven eight", 20)
        self.assertEqual(s2, "one two three...")
        self.assertLessEqual(len(s2), 20)

    # Exact-length text unchanged
    def test_exact_length_unchanged(self):
        self.assertEqual(truncate("USB-C dock", 10), "USB-C dock")


if __name__ == "__main__":
    unittest.main()
```

</details>