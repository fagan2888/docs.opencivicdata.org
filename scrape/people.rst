
.. _people:

Writing a Person Scraper
=========================

This document is meant to provide a tutorial-like overview of the steps toward contributing a munipal Person scraper to the Open Civic Data project.

This guide assumes you have a working pupa setup. If you don't please refer to the introduction on :doc:`/scrape/index`.


Special notes about People scrapers
-----------------------------------

The name is a bit misleading - so-called People scrapers actually scrape
``Person``, ``Organization`` and ``Membership`` objects.

The relationship between these three types is so close that they all should be
scraped at the same time.

Target Data
-----------

People scrapers pull in all sorts of information about ``Organization``
``Membership`` and ``Person`` objects.

The target data commonly includes:

* People, and their posts (what bodies they represent)

  * Alternate names
  * Current photo
  * links (homepage, YouTube account, Twitter account)
  * Contact information (email, physical address, phone number)
  * Any other identifiers that might be commonly used
  * Committee memberships

* Orgs (committees, etc)

  * Other names
  * Commonly used IDs
  * Contact information for the whole body
  * Posts (sometimes called seats) on the org
  * People in each org, and in which seat they sit.


Creating a New Person scraper
-----------------------------

Great. So, let's get started. Our person scraper can be located anywhere, and simply needs to be importable by the :file:`__init__.py` so that we can reference it in the `get_scraper` method. Your scraper can even by located in the :file:`__init__.py` file itself if you want to keep things extra simple, but scraper code can eventually get pretty lengthy, so its more scalable to break each scraper out into it's own file. Now open up the default :file:`people.py` scraper generated by the :program:`pupa` init program. It should look like this:

.. literalinclude:: ../../pupa/example/people.py

This is the default scraper template, which isn't very useful yet, but it helps
to clarify what the intent of the scraper is. Let's take a closer look.

Every `Person` scraper inherits a `scrape_people` method. Usually it's not
advised to override this method, rather, implementing a proper
`get_people` method (which will `yield` back `Person` objects to `scrape_people`
to save to disk) is the prefered way to write a scraper. Only in the most
extreme of cases should `scrape_people` be overridden.

You may also yield an iterable of `Person` objects, which helps if you
are scraping both people and committees for the Jurisdiction, but want
to keep the scraper logic in their own routines.

For more information on the ``scrape_people`` or ``get_people`` methods, you
might consider reading about
:meth:`pupa.scrape.base.Scraper.scrape_people` and
:meth:`pupa.scrape.base.Scraper.get_people` in the Pupa docs.

As you might have guessed by now, `Person` scrapers scrape many `People`, as
well as any `Membership` objects that you might find along the way.

Let's take a look at a dead-simple Pupa scraper::

    from pupa.scrape import Scraper, Legislator
    class MyFirstPersonScraper(Scraper):
        def get_people(self):
            js = Legislator(name="John Smith", post_id="Ward 1")
            js.add_source(url="http://example.com")
            yield js

You can see that we create the Legislator, with the only two required
params (`name` and `post_id`, add the source of the data (most of the time
this will be the url that you've called with `urlopen`) and yielded the
Legislator back.

Right. Now let's get back to Memberships. Let's say that we've found
that John Smith has a membership in the Transportation committee::

    from pupa.scrape import Scraper, Legislator
    class MyFirstPersonScraper(Scraper):
        def get_people(self):
            js = Legislator(name="John Smith", post_id="Ward 1")
            js.add_source(url="http://example.com")
            js.add_committee_membership("Transportation",
                                        role="Chair")
            yield js

Of course, all of this is well and good if we find all the data on the
same page. However, commonly, it's much easier to scrape each committee
from the committee pages, since this will often have the data in
an easier-to-scrape format.

Rather than write something like::

    from pupa.scrape import Scraper, Legislator, Committee
    class MyFirstPersonScraper(Scraper):
        def get_people(self):
            js = Legislator(name="John Smith", post_id="Ward 1")
            js.add_source(url="http://example.com")

            members = ["John Smith", "Jos Bleau"]

            committee = Committee("Transportation")
            committee.add_source("http://example.com/committee/transport")
            for member in members:
                committee.add_member(member, role='member')

            yield committee
            yield js

However, as you can imagine, this gets quite out of hand quite quickly. One
common pattern is to split the logic into two sections, such as::


    from pupa.scrape import Scraper, Legislator, Committee
    class MyFirstPersonScraper(Scraper):
        def scrape_legislators(self):
            js = Legislator(name="John Smith", post_id="Ward 1")
            js.add_source(url="http://example.com")
            yield js

        def scrape_committees(self):
            members = ["John Smith", "Jos Bleau"]
            committee = Committee("Transportation")
            committee.add_source("http://example.com/committee/transport")
            for member in members:
                committee.add_member(member, role='member')

            yield committee

        def get_people(self):
            yield self.scrape_legislators()
            yield self.scrape_committees()

It's worth noting that you should keep in mind `scrape_people` *is* a special
function (see above), so you should take care not to override this method.

Of course, in real scrapers, you'll need to write some code to take care
of getting the list of people that are in that jurisdiction, or have
memberships in the Legislature. Hardcoding names, such as in the examples
above is bad practice, since it's usually easier to just manually enter
in the data.

As a slightly more fun example, here's a scraper that will scrape the
Sunlight website for people's information. This is deliberately a mildly
complex example (as well as being purely for fun!), to get a feel for what
a working Person scraper may look like::

    import re
    import lxml
    from pupa.scrape import Scraper, Legislator, Committee
    class SunlightPersonScraper(Scraper):
        def get_people(self):
            url = "http://sunlightfoundation.com/people/"
            entry = self.urlopen(url)
            page = lxml.html.fromstring(entry)
            page.make_links_absolute(url)
            for person in page.xpath("//ul[contains(@class, 'sunlightStaff')]//li"):
                who = person.xpath(".//a")
                who = who[0] if who else None
                name = who.text_content().strip()
                image = person.xpath(".//div[@class='imgWrapper']/@style")
                image = image[0] if image else None
                if image:
                    urls = re.findall("url\((?P<url>.*)\);", image)
                position = person.xpath(".//span[@class='staffTitle']/text()")[0]
                position = position.strip()
                person = Legislator(name=name,
                                    post_id=position,
                                    image=image)
                person.add_link('homepage', who.attrib['href'])
                person.add_source(url)
                yield person


Special notes regarding Posts, Memberships and Districts
--------------------------------------------------------

The keen observer will note that we're using ``post_id`` to note the person's
primary position (for Legislative scrapers, this will usually be the member's
ward or district.

Looking at the `Popolo spec <http://popoloproject.com/>`_, you might be
confused on why this isn't an opaque ID, or some sort of slug.

We use full strings to help avoid second lookups and O(n) search-time from
a matching org when displaying the Post ID on something like a website (usually
in some sort of list).

By using the complete text of the post as the ID, we can better use
``Membership`` objects for display of the data. When we load a membership
obejct, we can just use the ``post_id`` as the display slug, without having
to load the ``Organization``, and iterate over the ``post`` objects until
we find a matching ``post_id``. We're still able to search globally based
on ``post_id`` by also constraining ``jurisdiction_id`` as well. In
addition, trying to slugify the ``post_id`` in the scraper is something
that might be quite error-prone anyway, so there isn't much of an upside
to trying to keep it as a pure opaque ID or slug.
