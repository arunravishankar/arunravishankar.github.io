---
category: Setup-Notes
---
# Setting Up Git Config Files for Sensitive Information
I was looking around for the best way to store API keys and other sensitive information that I use in my programs but don't want exposed to the public through my GitHub account. After doing a bit of digging into best practices [here](https://stackoverflow.com/questions/2397822/what-is-the-best-practice-for-dealing-with-passwords-in-github) and [here](https://stackoverflow.com/questions/21774844/proper-way-to-hide-api-keys-in-git) and [here](https://stackoverflow.com/questions/5272846/how-to-get-parameters-from-config-file-in-r-script), I decided to create a config file (`config.yml`) to store this information and then add the config file to the .gitignore file. 

Since the cofig file is not copied to the public repository, I also followed the guidance of creating an example config file (`config-sample.yml`) to illustrate what is needed.

I can then access the config variables within my script using the R package `yaml`. This example is from acs commute shiny app:

```
library(yaml)
config = yaml.load_file("config.yml")
acs_key <- config$acs$key
```

