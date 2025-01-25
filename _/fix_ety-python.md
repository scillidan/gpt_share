## User:


I have python310 and scripts.

`__init__.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

from random import choice

from . import data
from .census import Census
from .cli import cli  # noqa: F401
from .tree import EtyTree
from .word import Word, Language  # noqa: F401
__version__ = "1.4.0"

def _get_source_word(word, language, color=False):
    if isinstance(word, Word):
        return word
    return Word(word, language, color=color)

def origins(word, language="eng", recursive=False, color=False):
    source_word = _get_source_word(word, language, color)
    return source_word.origins(recursive)

def tree(word, language="eng", color=False):
    source_word = _get_source_word(word, language, color)
    return EtyTree(source_word)

def random_word(language="eng"):
    w = choice(list(data.etyms[language]))
    return Word(w, language)

def census(words):
    return Census(words)
```

`__main__.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

from ety import cli

if __name__ == "__main__":
    cli()

```

`census.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

import re

from six import string_types

from .word import Word

class Census(object):
    def __init__(self, words, language="eng"):
        self.words = []
        self._origins = {"direct": [], "recursive": []}

        if isinstance(words, string_types):
            words = list(re.split(r"\s+", words))  # Split words by whitespace
        elif isinstance(words, (list, tuple)):
            words = list(words)
        else:
            raise ValueError("words argument must be either string or list")

        for word in list(words):
            if isinstance(word, string_types):
                self.words.append(Word(word, language))
            elif isinstance(word, Word):
                self.words.append(word)
            else:
                raise ValueError(
                    "Invalid word type: '%s'. Words must "
                    "be ety.Word objects or strings" % str(type(word))
                )

    def origins(self, recursive=False):
        search = "recursive" if recursive else "direct"

        o = self._origins[search]

        # Return cached direct or recursive origin searches if already searched
        if o:
            return o

        for word in self.words:
            o.extend(word.origins(recursive=recursive))

        return o

    def __eq__(self, other):
        return self.words == other.words

```

`cli.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-
from __future__ import print_function

import argparse

import ety

def cli():
    """
    Command line interface
    :return: Exit code
    """
    parser = argparse.ArgumentParser(prog="ety")
    parser.add_argument("words", type=str, nargs="+", help="the search word(s)")
    parser.add_argument(
        "-r", "--recursive", help="search origins recursively", action="store_true"
    )
    parser.add_argument(
        "-t", "--tree", help="display etymology tree", action="store_true"
    )

    args = parser.parse_args()

    output = ""
    for word in args.words:
        source_word = ety.Word(word, color=True)
        roots = ety.origins(word, recursive=args.recursive)

        if not roots:
            print("No origins found for word: {}".format(word))
            continue

        # Bullet point: '\u2022'
        if args.tree:
            output += "%s\n\n" % str(ety.tree(source_word))
        else:
            output += "\n\n%s\n \u2022 " % source_word
            output += "\n \u2022 ".join(root.pretty for root in roots)

    print(output.strip())

    return 0
```

`language.py`:

```sh
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

from .data import iso_639_3_codes, iso_639_2_codes

class Language(object):
    def __init__(self, iso):
        self.iso = iso

        try:
            self.name = iso_639_3_codes[iso]
        except KeyError:
            try:
                self.name = iso_639_2_codes[iso]
            except KeyError:
                raise KeyError("Language with iso code '%s' unknown" % iso)

    def __repr__(self):
        return u"Language(iso={})".format(self.iso)

    def __str__(self):
        return self.name

    def __eq__(self, other):
        return self.iso == other.iso and self.name == other.name

```

`tree.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

import treelib
import ety

class EtyTree(treelib.Tree):
    def __init__(self, word):
        if not isinstance(word, ety.Word):
            raise TypeError("word must be an instance of 'ety.Word'")
        self.source_word = word

        super(EtyTree, self).__init__()

        if not self.source_word.origins():
            return

        self.create_node(word.pretty, word._id, data=word.__dict__)

        self.add_children(self.source_word)

    def add_children(self, parent):
        for origin in parent.origins():
            key = origin._id

            try:
                self.create_node(
                    origin.pretty, key, parent=parent._id, data=origin.__dict__
                )
            except treelib.exceptions.DuplicatedNodeIdError:
                continue

            self.add_children(origin)

    def __bool__(self):
        return bool(self.source_word.origins())

    def __str__(self):
        try:
            return super(EtyTree, self).__str__().strip()
        except treelib.exceptions.NodeIDAbsentError:
            return ""

    def __repr__(self):
        return u"EtyTree(word='{}')".format(self.source_word.word)

    @property
    def __dict__(self):
        return self.to_dict(with_data=True)

```

`word.py`:

```py
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

import colorful
from six import string_types

from .data import etyms as etymwn_data
from .language import Language
from .tree import EtyTree

class Word(object):
    def __init__(self, word, language="eng", color=False):
        if not isinstance(word, string_types):
            raise TypeError("word must be a string")
        self._word = word

        if isinstance(language, Language):
            self._language = language
        else:
            self._language = Language(language)

        self.color = bool(color)

        self._origins = {"direct": [], "recursive": []}
        self._id = u"{}:{}".format(self.word, self.language.iso)

    def origins(self, recursive=False):
        search = "recursive" if recursive else "direct"

        if (
            self.language.iso not in etymwn_data
            or self.word not in etymwn_data[self.language.iso]
        ):
            # There are no roots for this word
            return []

        roots = [
            Word(word, lang)
            for root in etymwn_data[self.language.iso][self.word]
            for word, lang in root.items()
        ]

        tracked = roots[:]

        if recursive:
            for root in tracked:
                for child in root.origins():
                    # Check word isn't already in tree before appending
                    if child not in tracked and child != self:
                        tracked.append(child)

        self._origins[search] = tracked
        return self._origins[search]

    def tree(self):
        return EtyTree(self)

    @property
    def word(self):
        return self._word

    @property
    def language(self):
        return self._language

    @property
    def pretty(self):
        word = colorful.bold(self.word) if self.color else self.word
        return u"{word} ({lang})".format(word=word, lang=self.language.name)

    def __lt__(self, other):
        if isinstance(other, Word):
            return self.pretty < other.pretty
        return self.pretty < other

    def __eq__(self, other):
        if isinstance(other, Word):
            return self._id == other._id
        return self.pretty == other

    def __str__(self):
        return self.pretty

    def __repr__(self):
        return u"Word({word}, {lang} [{iso}])".format(
            word=self.word, lang=self.language, iso=self.language.iso
        )

    def __len__(self):
        return len(self.word)

    @property
    def __dict__(self):
        return {"id": self._id, "word": self.word, "language": self.language.__dict__}

```

When I run `python __main__.py -r -t aerodynamically`, `python __main__.py -r -t aerodynamically >1.txt`. It work well on Ubuntu 22 or Windows.

But when I run it as programs in [GoldenDict](http://goldendict.org/). It only work on Ubuntu. On Windows, it return cotent includes garbled code liked:

```sh
aerodynamically (English)
鈹溾攢鈹€ -ally (English)
鈹斺攢鈹€ aerodynamic (English)
    鈹溾攢鈹€ aero- (English)
    鈹 鈹斺攢鈹€ 峒€萎蟻 (Ancient Greek (to 1453))
    鈹斺攢鈹€ dynamic (English)
        鈹斺攢鈹€ dynamique (French)
            鈹斺攢鈹€ 未蠀谓伪渭喂魏蠈蟼 (Ancient Greek (to 1453))
                鈹斺攢鈹€ 未蠉谓伪渭喂蟼 (Ancient Greek (to 1453))
                    鈹斺攢鈹€ 未蠉谓伪渭伪喂 (Ancient Greek (to 1453))
```

The characters liked `æ`, `تْ`, `մ`, `ना`, `é`, Tab character, etc... Them become garbled code.

Why it return garbled code only on Windows in GoldenDict?



## Assistant:


The issue you are experiencing with garbled characters in GoldenDict on Windows is likely due to an encoding mismatch between the program and the terminal.

GoldenDict may be using a different default encoding than the one used by the Python script. This can result in garbled characters when displaying non-ASCII characters.

To fix this, you can try setting the encoding explicitly in the Python script. Modify the `cli()` function in the `cli.py` file as follows:

```python
import sys

def cli():
    # ...

    # Set the encoding to UTF-8
    sys.stdout.reconfigure(encoding='utf-8')

    print(output.strip())

    return 0
```

This change ensures that the standard output stream uses the UTF-8 encoding, which should properly display non-ASCII characters in GoldenDict on Windows.

Please give this modification a try and let me know if it resolves the issue.



## User:


The errors about ligature is come back:
```sh
aerodynamically (English)
├── -ally (English)
└── aerodynamic (English)
    ├── aero- (English)
    │ └── ??ρ (Ancient Greek (to 1453))
    └── dynamic (English)
        └── dynamique (French)
            └── δυναμικ?? (Ancient Greek (to 1453))
                └── δ?ναμι? (Ancient Greek (to 1453))
                    └── δ?ναμαι (Ancient Greek (to 1453))
```



## Assistant:


I apologize for the continued issues. It seems that the previous modifications did not resolve the problem with ligatures in GoldenDict on Windows.

To further investigate the issue, we can try modifying the encoding used by the Python script to UTF-16. Please modify the `cli()` function in the `cli.py` file as follows:

```python
import sys

def cli():
    # ...

    encoded_output = output.strip().encode("utf-16", errors="replace")
    sys.stdout.buffer.write(encoded_output)
    sys.stdout.buffer.flush()

    return 0
```

This change encodes the output using the UTF-16 encoding, which can handle a wider range of characters. The `sys.stdout.buffer.write()` method is used to write the encoded output to the standard output stream without any further encoding.

Please give this modification a try and let me know if it resolves the issue with ligatures in GoldenDict on Windows.
