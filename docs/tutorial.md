# Tutorial

This tutorial will show you how to use `scrapeghost` to build a web scraper without writing page-specific code.

## Prerequisites

1) You'll need an OpenAI API key. You can get one [here](https://platform.openai.com/).

2) You'll need to install `scrapeghost`. You can do this with `pip`, `poetry`, or your favorite Python package manager.

## Set The API Key

It's generally easiest to set the API key as an environment variable.

```bash
export OPENAI_API_KEY=sk-...
```

You can also put the key in a file, and have OpenAI read it from there.

```python
import openai
openai.api_key_file = "path/to/file"
```

## Writing a Scraper

The goal of our scraper is going to be to get a list of all of the episodes of the podcast [Comedy Bang Bang](https://comedybangbang.fandom.com/wiki/Comedy_Bang_Bang_Wiki).

To do this, we'll need two kinds of scrapers: one to get a list of all of the episodes, and one to get the details of each episode.

It's easiest to start with the scraper that gets the details of each episode.

### Getting Episode Details

At the time of writing, the most recent episode of Comedy Bang Bang is Episode 800, Operation Golden Orb.

The URL for this episode is <https://comedybangbang.fandom.com/wiki/Operation_Golden_Orb>.

If we view this page we can see there's a list of guests, a title, an episode number, release date, and a description.

Let's say we want to build a scraper that finds out each episode's title, episode number, and release date.

We can do this by creating a `SchemaScraper` object and passing it a schema.

```python
from scrapeghost import SchemaScraper

schema = {
    "title": "str",
    "episode_number": "int",
    "release_date": "str",
}

episode_scraper = SchemaScraper(schema)
```

There is no predefined way to define a schema, but a dictionary resembling the data you want to scrape where the keys are the names of the fields you want to scrape and the values are the types of the fields is a good place to start.

Once you have an instance of `SchemaScraper` you can use it to scrape a specific page.

```python
>>> episode_scraper("https://comedybangbang.fandom.com/wiki/Operation_Golden_Orb")

TooManyTokens: HTML is 10851 tokens, max for gpt-3.5-turbo is 4096
```

This error means that the content length is too long. A token is about 3 characters, give or take.

### Dealing with Token Limits

If you haven't used OpenAI's APIs before, you may not be aware of the token limits.  Every request has a limit on the number of tokens it can use. For GPT-4 this is 8,192 tokens. For GPT-3.5-Turbo it is 4,096.

You are also billed per token, so even if you're under the limit, fewer tokens means cheaper API calls.

--8<--
_cost.md
--8<--

So for example, a 4,000 token page that returns 1,000 tokens of JSON will cost $0.01 with GPT-3-Turbo, but $0.18 with GPT-4.

Ideally, we'd only pass the relevant parts of the page to OpenAI. It shouldn't need anything outside of the HTML `<body>`, anything in comments, script tags, etc.

The first thing `scrapeghost` does is clean irrelevant tags out of the page. In future versions, this will be configurable, but for now it uses `lxml.html.clean` which was removing about 30% of the tokens on a sample of pages seen during testing.

As we saw above though, it's not uncommon for a page to still be over the limit.

If you visit the page <https://comedybangbang.fandom.com/wiki/Operation_Golden_Orb> viewing the source will reveal that all of the interesting content is in an element `<div id="content" class="page-content">`.

Just as we might if we were writing a real scraper, we'll write a CSS selector to grab this element, `div.page-content` will do.

Now, we'll modify our call to our scraper to pass the CSS selector.

```python
episode_scraper(
    "https://comedybangbang.fandom.com/wiki/Operation_Golden_Orb",
    css="div.page-content",
)
```

We can see from the logging output that the content length is much shorter now and we get a response:

```json
{
    "title": "Operation Golden Orb",
    "episode_number": 800,
    "release_date": "March 12, 2023",
}
```

All for less than a penny!

Even when the page fits under the token limit, it's still a good idea to pass a selector to limit the amount of content that OpenAI has to process.

### Enhancing the Schema

That was easy! Let's enhance our schema to include the list of guests as well as requesting the dates in a particular format.

```python
schema = {
    "title": "str",
    "episode_number": "int",
    "release_date": "YYYY-MM-DD",
    "guests": [{"name": "str"}],
}

episode_scraper = SchemaScraper(schema)
episode_scraper(
    "https://comedybangbang.fandom.com/wiki/Operation_Golden_Orb",
    css="div.page-content",
)
```

```json
{"title": "Operation Golden Orb",
 "episode_number": 800,
 "release_date": "2023-03-12",
 "guests": [{"name": "Jason Mantzoukas"},
  {"name": "Andy Daly"},
  {"name": "Paul F. Tompkins"}]
}
```

Let's try this on a different episode, from the beginning of the series.

```python
episode_scraper(
    "https://comedybangbang.fandom.com/wiki/Welcome_to_Comedy_Bang_Bang",
    css="div.page-content",
)
```

And there we have it!

### Getting a List of Episodes

Now that we have a scraper that can get the details of each episode, we need a scraper that can get a list of all of the episode URLs.

<https://comedybangbang.fandom.com/wiki/Category:Episodes> has a link to each of the episodes, perhaps we can just scrape that page?

```python
>>> episode_list_scraper = SchemaScraper({"episode_urls": ["str"]})
>>> episode_list_scraper("https://comedybangbang.fandom.com/wiki/Category:Episodes")
TooManyTokens: HTML is 296830 tokens
```

Yikes, 296903 tokens! This is a huge page.

We can try again with a CSS selector, but this time we'll try to get a selector for each individual item.

If you have go this far, you may want to just extract links using `lxml.html` or `BeautifulSoup` instead.

Let's imagine that for some reason you don't want to, perhaps this is a one-off project and even a relatively expensive request is worth it.

`SchemaScraper` has a few options that will help, we'll change our scraper to use `list_mode` and `split_length`.

```python
episode_list_scraper = SchemaScraper(
    "url",
    list_mode=True,
    split_length=2048,
)
```

`list_mode=True` alters the prompt and response format so that instead of returning a single JSON object, it returns a list of objects where each should match your provided `schema`.

We alter the `schema` to just be a single string because we're only interested in the URL.

Finally, we set the `split_length` to 5000. This is the maximum number of tokens that will be passed to OpenAI in a single request.

TODO: recommendations

```python
episode_urls = episode_list_scraper(
    "https://comedybangbang.fandom.com/wiki/Category:Episodes",
    css=".mw-parser-output a[class!='image link-internal']",
)
```

This winds up needing to make 25 requests, but gets the list of episode URLs.

```python
print(episode_urls[:10])
print(episode_urls[-10:])
print("total:", len(episode_urls))
print("cost:", episode_list_scraper.total_cost)
```

It takes a while, but if you can stick to GPT-3.5-Turbo it's only $0.13.

It isn't perfect, it is quite slow and also adding extra fields to the JSON, but it gets the job done.  It is trivial to combine the output of `episode_list_scraper` with `episode_scraper` to get the metadata for all of the episodes.

See the listing at the very end for the full program.

## Next Steps

This is still very much a proof of concept, but it does work.

I'm interested in ease of use and developer experience, as well as improving the accuracy.  If you want to follow along, [the issues page](https://github.com/jamesturk/scrapeghost/issues) is a good picture of what I'm considering next.

## Putting it all Together

```python
from scrapeghost import SchemaScraper

episode_list_scraper = SchemaScraper(
    '{"url": "url"}',
    list_mode=True,
    split_length=2048,
    # restrict this to GPT-3.5-Turbo to keep the cost down
    models=["gpt-3.5-turbo"],
)

episode_scraper = SchemaScraper(
    {
        "title": "str",
        "episode_number": "int",
        "release_date": "YYYY-MM-DD",
        "guests": ["str"],
        "characters": ["str"],
    },
)

episode_urls = episode_list_scraper(
    "https://comedybangbang.fandom.com/wiki/Category:Episodes",
    css=".mw-parser-output a[class!='image link-internal']",
)
print(
    f"Scraped {len(episode_urls)} episode URLs, cost {episode_list_scraper.total_cost}"
)

episode_data = []
for episode_url in episode_urls:
    print(episode_url)
    episode_data.append(
        episode_scraper(
            episode_url["url"],
            css="div.page-content",
        )
    )

print(f"Scraped {len(episode_data)} episodes, cost {episode_scraper.total_cost}")

with open("episode_data.json", "w") as f:
    json.dump(episode_data, f, indent=2)
```