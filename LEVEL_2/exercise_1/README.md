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
  U16_GET(text, 0, result.start, result.length, first_character);  [1]

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
==========================================================
#define CHECK_LE(val1, val2) CHECK_OP(<=, val1, val2)
  ```
  [1] call `U16_GET` after `DCHECK_LE`. This check means `result.start + result.length` must lessthan `text_length`, we can see about `U16_GET`
  ```c++
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
            if((i)+1!=(length) && U16_IS_TRAIL(__c2=(s)[(i)+1])) { \ [2]
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
  the third parameter is the length of the target string which be searched, just like find xy in xyd, and the length of this time is two.
  But [2] makes me puzzle, it seems like the `length` parameter is the end index of the `xyd`, but in truth it is the length of `xy`. And `@param length string length` proves my opinion. If we assignment i == length like i = 2, length = 2 and `__c2=(s)[(i)+1]` can oob read.
  We can check our answer by Detail.

  > This patch chagnes |IsWholeWordMatch()| to use |U16_GET()| with valid
  > parameters to avoid reading out of bounds data.
  >
  > In case of search "\uDB00" (broken surrogate pair) in "\u0022\uDB00", we
  > call |U16_GET(text, start, index, length, u32)| with start=1, index=1,
  > length=1, where text = "\u0022\DB800", then |U16_GET()| reads text[2]
  > for surrogate tail.
  >
  > After this patch, we call |U16_GET()| with length=2==end of match, to
  > make |U16_GET()| not to read text[2].



</details>

--------
