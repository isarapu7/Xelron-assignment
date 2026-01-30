# Part 3: Prompt Preparation

## Selected Pull Request

**PR #3883: Experimental "bare-ASCII" matching query**  
https://github.com/beetbox/beets/pull/3883

---

## 3.1 Repository Context 

Beets is basically for people who are obsessive about their music libraries. It's a command-line tool (though there are plugins for web interfaces and stuff) that helps you organize and manage your digital music collection. The folks using it are usually pretty technical - think developers, sysadmins, power users who aren't afraid of the terminal.

The problem it solves is: you've got thousands of music files from different sources (ripped CDs, downloads, whatever) and they're a complete mess. Tags are wrong, album art is missing, file names are inconsistent. Some tracks say "Unknown Artist - Track 01.mp3" even though you know exactly what they are. Fixing this manually would take forever and be incredibly tedious.

Beets automates all that. It talks to MusicBrainz (a huge music metadata database) to figure out what each track actually is and tags it correctly. But beyond just auto-tagging, it's got this flexible query system so you can search and manipulate your library however you want. And then there's the plugin system - that's the killer feature. You can extend beets to do pretty much anything with your music collection without touching the core code.

Most people interact with beets through commands like "beet import" to add music, "beet list" to search, "beet modify" to change things. It's all text-based but super powerful once you get used to it.

This PR we're looking at is all about the query system. Right now if you want to search for something, you need to type it exactly as it appears in your library. Got an album by "Björk"? You need to type that ö. This PR makes it so you can just type "bjork" with regular letters and still find it.

---

## 3.2 Pull Request Description 

So this PR adds a new plugin called "bareasc" that lets you search for stuff without typing special characters. The way it works: you put a hash mark (#) before your search term, and beets will match it against Unicode characters by converting everything to plain ASCII first.

New plugin file (`beetsplug/bareasc.py`) with the main query class. It hooks into Unidecode (a library that does Unicode-to-ASCII conversion) and compares the converted versions instead of the raw strings. So when you type `beet ls #bjork`, it converts "Björk" in your database to "Bjork" and checks if "bjork" matches. There's also a diagnostic command so you can see what the transformations look like - super helpful for debugging why something did or didn't match.

The documentation explains what this solves and why you'd want it. Configuration is simple - you can even change the prefix if # doesn't work for you. Tests cover different character sets and edge cases.

This was needed because the old way was frustrating as hell. You've got correctly formatted metadata (which is good! you want those proper accents and umlauts), but searching required typing those exact characters. Try doing that on a phone keyboard when you're looking for that German metal band with the umlauts. Or you're using an English keyboard layout and need to search for French artists. You'd either hunt for the special character to copy-paste, change your keyboard layout temporarily, or just scroll through your whole library manually.

Before: exact match only. "bjork" finds nothing if your library has "Björk".
After: if you use the # prefix, "bjork" finds "Björk". 

Important: this is one-way. ASCII queries find Unicode data, but not backwards. And it doesn't touch your stored data at all - transformations only happen during the search itself. If you don't enable the plugin or don't use the prefix, nothing changes from how queries work now.

---

## 3.3 Acceptance Criteria 

✓ When a user enables the bareasc plugin and performs a query with the "#" prefix, the system should successfully match database entries containing accented characters using plain ASCII search terms

✓ The implementation should preserve all existing query functionality - queries without the "#" prefix should behave exactly as they did before the plugin was added

✓ The plugin should handle a comprehensive range of Unicode transformations including European accents (é, ä, ö, etc.), special characters (ß, ñ), and other common non-ASCII characters that have ASCII equivalents

✓ When searching with "#motorhead", the system should find "Motörhead", and when searching with "#bjork" it should find "Björk", demonstrating case-insensitive matching combined with character transformation

✓ The implementation should support customization of the query prefix through beets' configuration system, allowing users to choose an alternative prefix if "#" conflicts with their existing queries or preferences

✓ The plugin should work correctly with both item-level queries (individual tracks) and album-level queries, applying transformations consistently across different types of library objects

✓ A diagnostic command should be available (such as "beet bareasc") that shows users how their query terms will be transformed, helping them understand and verify the matching behavior

✓ The implementation should maintain compatibility with both Python 2 and Python 3, handling Unicode strings appropriately in both environments without errors

✓ Database metadata should never be modified by this plugin - all transformations should occur only during query evaluation, ensuring data integrity is preserved

✓ The plugin should include comprehensive unit tests covering various Unicode character sets, edge cases, and both successful and failed match scenarios

---

## 3.4 Edge Cases (Minimum 3 cases)

**Edge Case 1: Multiple Diacritics on Single Characters**
Some characters have multiple diacritical marks or complex Unicode representations. For example, Vietnamese characters like "ớ" (o with horn and acute accent) or combined characters. The implementation should handle these by properly transforming them through Unidecode rather than failing or producing unexpected results. Test with artist names from languages using complex character combinations.

**Edge Case 2: Characters with No ASCII Equivalent**
Certain Unicode characters, particularly from non-Latin scripts (Cyrillic, Arabic, Chinese, Japanese, Korean), may not have direct ASCII equivalents. The system should handle these gracefully by either transliterating them sensibly via Unidecode or allowing the query to fail gracefully without crashing. For example, searching for "#Пушкин" (Russian) or "#北京" (Chinese) should either produce appropriate matches or clear "no results" without errors.

**Edge Case 3: Empty or Whitespace-Only Queries**
When a user types just the prefix character "#" with no following text or only whitespace, the system should handle this appropriately - either by treating it as an invalid query with a helpful error message or by falling back to standard query behavior. It should not crash or produce unexpected database queries.

**Edge Case 4: Special Characters That Are Both Query Syntax and Data**
Some characters might be used in beets' query syntax (like colons for field specification or quotes for exact matching) and also appear in Unicode transformations. The system should properly distinguish between characters that are part of the query language syntax versus characters that are part of the search term being transformed. For example, "#artist:bjork" should properly parse "artist" as a field specifier while transforming "bjork" for matching.

**Edge Case 5: Performance with Very Large Libraries**
Since transformation happens at query time for every potential match, very large libraries (100,000+ tracks) could experience performance degradation. The system should remain responsive and complete queries in reasonable time, even for broad searches that require checking many items. Testing should verify that query time remains acceptable (under 5-10 seconds for typical broad searches on large libraries).

---

## 3.5 Initial Prompt 

You are tasked with implementing a new query plugin for the beets music library management system. This plugin will enable users to search their music library using plain ASCII characters while matching against metadata containing Unicode characters like accents and special symbols.

**Context and Background:**
Beets uses a plugin-based architecture where query functionality can be extended through custom query classes. The query system processes user input and matches it against database fields containing music metadata (artist names, album titles, track names, etc.). Currently, searching requires exact character matching, meaning a user must type "Motörhead" to find that artist - typing "Motorhead" would fail.

**Your Task:**
Create a new plugin called "bareasc" that implements the following functionality:

1. A custom query class that extends beets' base query system
2. A configurable prefix character (default "#") that triggers ASCII-normalized matching
3. Integration with the Unidecode library to transform both query input and database values to ASCII equivalents
4. A diagnostic command showing how transformations will appear to users

**Implementation Requirements:**

The plugin should be placed in `beetsplug/bareasc.py` and implement a query class that:
- Registers the "#" prefix (or user-configured alternative) with beets' query parser
- When the prefix is detected, transforms both the search term and database field values using Unidecode
- Performs case-insensitive substring matching on the transformed values
- Returns matches when the ASCII-normalized query is contained in the ASCII-normalized database value

**Acceptance Criteria to Address:**
- Successfully match "Motörhead" when searching for "#motorhead"
- Successfully match "Björk" when searching for "#bjork"  
- Work with both item and album queries
- Never modify the stored metadata - transformations occur only during matching
- Support Python 2 and Python 3 with proper Unicode handling
- Allow users to configure the prefix character through beets' configuration system
- Provide a "beet bareasc" command that displays transformed output for debugging

**Edge Cases to Consider:**
- Characters with multiple diacritics (Vietnamese, complex European characters)
- Non-Latin scripts (Cyrillic, Arabic, CJK) that may not have ASCII equivalents
- Empty or whitespace-only queries after the prefix
- Interaction between the prefix and other query syntax elements
- Performance implications on large libraries (100,000+ tracks)

**Testing Requirements:**
Create comprehensive unit tests in `test/test_bareasc.py` covering:
- Various Unicode character transformations
- Both successful and failed matches
- Item and album query types
- Edge cases listed above
- Python 2 and 3 compatibility

**Documentation Requirements:**
Create `docs/plugins/bareasc.rst` including:
- Clear explanation of the feature and its use cases
- Configuration instructions
- Query examples
- Credit to the Unidecode library
- Add the plugin to the plugins index

**Technical Considerations:**
- Use the existing beets plugin registration system
- Follow beets' coding conventions and style
- Ensure backward compatibility - non-prefixed queries should work exactly as before
- Keep transformations stateless and pure (no side effects)
- Consider query performance and optimize where possible

Deliver a working implementation with tests and documentation that integrates seamlessly with the existing beets codebase.

---

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
