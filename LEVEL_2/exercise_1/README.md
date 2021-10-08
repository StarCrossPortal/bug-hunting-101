# Exercise 1

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21128
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

In level 2, we do it without the help of Details



---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>
    
  https://bugs.chromium.org/p/chromium/issues/detail?id=1138877

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset -hard 04fe9cc9bf0b67233b9f7f80b9a914499a431fa4
```

### Related code

[third_party/blink/renderer/core/editing/iterators/text_searcher_icu.cc](https://chromium.googlesource.com/chromium/src/+/04fe9cc9bf0b67233b9f7f80b9a914499a431fa4/third_party/blink/renderer/core/editing/iterators/text_searcher_icu.cc)


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  `IsWholeWordMatch` This func looks like buggy.
  ```c++
static bool IsWholeWordMatch(const UChar* text,
                             int text_length,
                             MatchResultICU& result) {
  DCHECK_LE((int)(result.start + result.length), text_length);
  UChar32 first_character;
  U16_GET(text, 0, result.start, result.length, first_character);

  // Chinese and Japanese lack word boundary marks, and there is no clear
  // agreement on what constitutes a word, so treat the position before any CJK
  // character as a word start.
  if (Character::IsCJKIdeographOrSymbol(first_character))
    return true;

  wtf_size_t word_break_search_start = result.start + result.length;
  while (word_break_search_start > result.start) {
    word_break_search_start =
        FindNextWordBackward(text, text_length, word_break_search_start);
  }
  if (word_break_search_start != result.start)
    return false;
  return static_cast<int>(result.start + result.length) ==
         FindWordEndBoundary(text, text_length, word_break_search_start);
}
=======================================================================
/**
 * Get a code point from a string at a random-access offset,
 * without changing the offset.
 * "Safe" macro, handles unpaired surrogates and checks for string boundaries.
 *
 * The offset may point to either the lead or trail surrogate unit
 * for a supplementary code point, in which case the macro will read
 * the adjacent matching surrogate as well.
 *
 * The length can be negative for a NUL-terminated string.
 *
 * If the offset points to a single, unpaired surrogate, then
 * c is set to that unpaired surrogate.
 * Iteration through a string is more efficient with U16_NEXT_UNSAFE or U16_NEXT.
 *
 * @param s const UChar * string
 * @param start starting string offset (usually 0)
 * @param i string offset, must be start<=i<length
 * @param length string length
 * @param c output UChar32 variable
 * @see U16_GET_UNSAFE
 * @stable ICU 2.4
 */
#define U16_GET(s, start, i, length, c) UPRV_BLOCK_MACRO_BEGIN { \
    (c)=(s)[i]; \
    if(U16_IS_SURROGATE(c)) { \
        uint16_t __c2; \
        if(U16_IS_SURROGATE_LEAD(c)) { \
            if((i)+1!=(length) && U16_IS_TRAIL(__c2=(s)[(i)+1])) { \
                (c)=U16_GET_SUPPLEMENTARY((c), __c2); \
            } \
        } else { \
            if((i)>(start) && U16_IS_LEAD(__c2=(s)[(i)-1])) { \
                (c)=U16_GET_SUPPLEMENTARY(__c2, (c)); \
            } \
        } \
    } \
} UPRV_BLOCK_MACRO_END
  ```


</details>

--------
