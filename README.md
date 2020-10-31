Convert Rich Text Files to Markdown on macOS
================================================================================

I had quite a few .rtf note files I wanted to convert to (mostly) properly formatted markdown - which this script does. Also, my notes had a lot of inline code snippets as well as larger code sections. The code in these notes was always marked in bold for readability, so that's used to identify code in the conversion to markdown if you specify the `--codenotes` flag. 

#### Usage
    convertRtfToMarkdown <src dir> <dst dir> [--codenotes]

E.g. to convert all .rtf files in the current directory to markdown versions in the md subdirectory:

    <path>/convertRtfToMarkdown . ./md

pdanford - Oct 2020

--------------------------------------------------------------------------------

`convertRtfToMarkdown` converts all .rtf files in a directory to standard markdown saved to a destination directory using macOS's **textutil** and **sed**, plus **pandoc**. Pandoc can be installed in macOS with Homebrew: `brew install pandoc`

The markdown generated is a pretty clean markdown version of the Rich Text but markdown compliance depends on the source files not being at odds with markdown syntax. For instance, a word in the rtf written like \*\*word\*\* (and not bolded) will appear as bold in markdown (i.e. word is surrounded by asterisks) if `--codenotes` was specified.

But if `--codenotes` is not specified, \*\*word\*\* in .rtf will appear in the markdown escaped as \\\*\\\*word\\\*\\\* and therefore render as it did in the .rtf (as non-bolded word with asterisks). Moreover, in this mode the characters ``]\`*_#[`` appearing in the rtf (that have special meaning in markdown) are escaped so they appear literally when rendered.

Since Markdown doesn't have a defined syntax to underline text for some odd reason, underlined words in rtf appear as `<u>word</u>` in the markdown. And other things such as `---` and `===` will still render as dividers or headings in both modes.

So if emphasized words are to be correctly converted to markdown, they should actually be rtf **bold** or *italics* so they convert to markdown correctly as \*\*bold\*\* and \*italic\* (i.e. the rtf version doesn't use asterisks to denote attributes - but rather the normal rtf way of bolding text). And note that if `--codenotes` is specified, rtf bolded words/sentences are further transformed along with other characters that have special meaning in markdown (see next section).

### When the rtf source is comprised of notes mixed with code and cli examples
The code notes stage (performed when the `--codenotes` is specified as the 3rd script argument) further transforms the markdown to make it better mirror the rtf when the rtf comprises notes mixed with command line or code examples. Anything marked as bold in the rtf is converted to markdown code blocks or inline code blocks. Also, the escapes used for markdown special characters (e.g. \_\#\*) are unescaped so they appear unaltered in code blocks (see note 1 below). That is, all markdown special characters are unescaped in this mode because it's assumed they will only appear in code blocks.

#### sed pattern explanations
textutil and pandoc aren't quite enough to do a complete conversion of rtf to markdown. Here are quick explanations of the additional manipulation done by sed:

    's/(<span)([^>]*>)([^<]*)(<\/span>)/\3/g' -> remove textutil's space spanning strangeness
    's/&lt;/</g'                              -> convert html escaped less than to actual character
    's/&gt;/>/g'                              -> convert html escaped greater than to actual character
    's/(^[0-9]+)\)/\1./g'                     -> convert any numbered lists that use #) to markdown's dotted syntax #.
    's/ / /g'                                 -> convert any UTF8 non-breaking space to regular space (see note 2 below)

#### --codenotes stage sed pattern explanations

    'N;s/(^[[:space:]]*[^* ].*)(\n)([[:space:]]*\*\*)/\1\2\2\3/;t' -e 'P;D;' -> add a blank line after each occurrence
                                                                                of a line that starts without bold rtf
                                                                                and the next line that is bold (i.e.
                                                                                valid md code blocks need a blank line
                                                                                before the block start)
    's/(^[[:space:]]*)(\*\*)(.*)(\*\*)(.*)/    \1\3\5/'                      -> and indent the lines that start with rtf
                                                                                bold and remove the markdown ** marks
                                                                                (these two make a valid stand-alone md
                                                                                code block)
    's/(\*\*)([^*]+)(\*\*)/`\2`/g'        -> transform inline rtf bold areas into markdown inline code
                                             blocks (not greedy)
    's/(\*\*)([^*`])(.*)(\*\*)/`\2\3`/g'  -> transform inline rtf bold areas into markdown inline code
                                             blocks (lines missed by above because of single asterisk
                                             inside)
    's/(\\)([]\`*_#[])/\2/g'              -> remove Pandoc inserted backslash escapes (see note 1 below)
    's/`([[:space:]]+)`/\1/g'             -> un-inline code spans that contain only spaces; these happen
                                             when edits are made to bolded sections in the rtf

#### Notes:
1. Pandoc inserted backslash escapes are removed so the markdown version matches rtf more closely in `--codenotes` mode. This is so things like filenames in the rtf that have under-bars or command examples that have asterisks will not be altered with the \ escape. Command and file paths in the markdown will match verbatim with how they appear in the rtf. These being unescaped won't interfere with markdown if they only occur in bolded areas in the rtf since they will appear in code blocks in the markdown version.
2. The reason the regex is not 's/\xC2\xA0/ /g' is so this will work macos default BSD sed. So beware the 's/ / /g' is actually two different characters if you xxd it (this is a utf-8 file).
3. `<a>` and `<p>` are suppressed in textutil so the output more closely matches the way the rtf source looks.

---
:scroll: [MIT License](README.license)
