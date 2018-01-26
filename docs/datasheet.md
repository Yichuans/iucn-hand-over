# Introduction

Throughout the past four decades UNEP-WCMC has inherited [a legacy of datasheets](http://old.unep-wcmc.org/world-heritage-information-sheets_271.html), records that cover consistently a wide variety of topics for each and every natural World Heritage site. While still available in the new website as [information sheets](https://www.unep-wcmc.org/resources-and-data/world-heritage-information-sheets), such valuable information is fragmented, and their usefulness has been severely limited by the lack of access.

> If we make the datasheets hard to find, people will not use them

The overhaul of datasheets has been ongoing since I joined the World Heritage Programme in 2011. However, it is not until late 2017, with the help of many interns, this work finally concludes as a prototype, and made accessible online. 

# Challenge

The two distinct challenges are to 1) improve accessibility by publishing the datasheets online, preferably with good search capabilities; and 2) convert the ageing word/pdf format to an easily maintainable and publishable format.

The datasheets in the word format compromise more than decades of editing and evolving styles. They are inherently inconsistent, to say the very least. Stripping off formatting and then structuring them is no small task. 

# Presentation

From a user point of view, I feel the traditional dynamic website may be an over-engineered option. The content should remain largely unchanged throughout the year and that interactivity is not essential. In fact, all pages can be cached. With this in mind, I decided the system to serve datasheets shall have no database, no web server and thus static. 

This thinking bears resemblance of blogging systems, in which articles are written in a pure text format and the web pages are generated using a [static site generator](https://en.wikipedia.org/wiki/Static_web_page). It offers simplicity, superb scalability and above all speed. 

The challenge is to somehow adapt the tool to generalise the specific use case of blog to more generic data. I used [Pelican](http://docs.getpelican.com/en/stable/), a Python powered static site generator, for this purpose. [This repository](https://github.com/Yichuans/datasheet) illustrates how I did it.

# Content

Converting and cleaning the content needs a sustainable solution. This is because UNEP-WCMC updates datasheets annually, either adding new sites or modifying those with significant changes (for example, extension). The process will likely run at least once a year.

Here are the steps:

1. Use [Pandoc](http://pandoc.org) to batch convert word format to markdown format (1st batch)
2. Proof read the converted markdown format to check for systematic issues and unique problems within individual datasheets
3. Programmatically fix systematic issues in the markdown format (notably using regular expressions) and on the spot fixes in the original word format if deemed one off and easier
4. Programmatically extract information in the markdown format and append them at the beginning for the final export of datasheets in markdown format (2nd batch). This is required by the Pelican site generator.

See [this Github repository](https://github.com/Yichuans/datasheet-format-pelican-md) for more details.
