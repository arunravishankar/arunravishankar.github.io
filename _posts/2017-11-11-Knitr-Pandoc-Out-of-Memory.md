---
category: Blog
---
# Windows Pandoc 'Out of Memory' Error
I recently started using R Notebook and was having problems with `knitr` to save as a markdown file. On Windows, Rmarkdown uses pandoc to render Rmd files, and I kept getting an ‘out of memory’ error even though I could see in the Task Manager that I still had more memory available. The issue was not a bug; apparently Windows sets an artificial limit of 2GB RAM allowed by the pandoc.exe application. I found the best explanation and solution on [Jonathan Chang's Blog](https://jonathanchang.org/blog/fixing-pandoc-out-of-memory-errors-on-windows/).

Rather than going with the Microsoft Visual Studio solution, I folllowed his link to the lightweight GUI [Address Aware Application](https://www.techpowerup.com/forums/threads/large-address-aware.112556/). It is really easy to use--simply download 'la_2_0_4.zip', run the application, find pandoc.exe, check the box to allow more than 2GB, then restart the computer and you are good to go!