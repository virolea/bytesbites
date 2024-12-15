---
layout: post
title:  "Going native with ActionMailbox"
excerpt: "ActionMailbox has the most underrated fun-to-feature ratio. No suprise here then, that as soon as I had the chance to work with it on Laterbox, I jumped on the occasion."
image: "assets/images/social-cards/going-native-with-actionmailbox.png"
author: Vincent Rolea
tags: rails actionmailbox laterbox email
---


# Going native with ActionMailbox

Concurrently to my [recent job search](https://x.com/vincentrolea/status/1841110773336068339), I embarked on a new hobby project called [Laterbox](https://laterbox.io). It’s a web application where one can bookmark all the content - blog articles, videos, websites -  one finds online, and have an inbox of sorts (namely the laterbox) to group all resources one has yet to checkout.

![Preview of Laterbox interface](/assets/images/going-native-with-actionmailbox/laterbox-preview.png)

The idea is not new, and there’s already a ton of existing tools out there that achieve a similar goal.  However none of them clicked with me, so I decided to build my own. Plus it was a perfect excuse to start a new Rails project, and test all the cool new features the eighth major version of the framework had to offer: Authentication template ([more about it here](https://bytesbites.io/2024/11/13/authentication-in-rails-8-jam-sessions.html)), the Solid Trifecta, production SQLite support and Kamal 2 to name the main ones.

But the subject of this article is not a shiny new Rails 8 library, rather a grandfather of the framework introduced in Rails 6: [ActionMailbox](https://edgeguides.rubyonrails.org/action_mailbox_basics.html). To the neophyte, ActionMailbox helps with inbound emails, by handling incoming webhooks from your email service, parsing and storing emails and offering a controller-like structure to handle the work to be done.

In my opinion, ActionMailbox has the most underrated fun-to-feature ratio. Some have already demonstrated it already, either by [starting up their truck](https://www.youtube.com/watch?v=i-RwxAVMP-k) by the send of an email, or literaly [setting emails on fire](https://signalvnoise.com/svn3/the-making-of-a-dumpster-fire/). This is a neat little library that can do **a LOT**. No suprise here then, that as soon as I had the chance to work with it on Laterbox, I jumped on the occasion.

## The feature

Laterbox helps with bookmarking resources you find online. The main entrypoint to the app is the resource form, used to save a new resource in your laterbox:

![Laterbox new resource form](/assets/images/going-native-with-actionmailbox/new-resource-form.png)

This works really well for in-app experience, however it does not fit the “real” flow of discovering new content online. Most content is found either in another app, in an email newsletter, or browsing the internet on your phone. Having to copy paste the link between apps make this quite tedious and don’t add up to a great experience. The answer, as you might have guessed it, would be to go mobile, and build a native share feature ([ShareExtension on iOS](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/Share.html) or [Sharesheet on Android](https://developer.android.com/training/sharing/send)), similarly to the apps featured in the menu below on iOS:

![iOS Sharing preview](/assets/images/going-native-with-actionmailbox/share-preview.png)

Hotwire native seems promising, but I am guessing there’s a bit of a learning curve here and I don’t have the bandwith available to build an experiste of the matter. Looking more closely at the available apps in the share menu, I realised something: what if I could use my email application to save resources on Laterbox? *ActionMailbox has entered the room*,

## Building the feature

**Note**: I won’t cover ActionMailbox configuration for your email service, as it’s pretty well covered in the official guides and other blog articles and conference talks.

Each user in Laterbox gets an inbound email address, of the form  `username_eufdnt45@inbound.laterbox.io`. This is the email address that needs to be sent to when sharing a resource via email:

![Screenshot from mail application](/assets/images/going-native-with-actionmailbox/in-app-mail.png)

Every email sent to `@inbound.laterbox.io` is routed to a dedicated `InboundsMailbox`:

```ruby
# app/mailboxes/application_mailbox.rb

class ApplicationMailbox < ActionMailbox::Base
  routing /@inbound\./i => :inbounds
end

```

The Inbounds mailbox’ role is to retrieve the user for the corresponding inbound email address, extract urls from the body and create resources if there are any:

```ruby
# app/mailboxes/inbounds_mailbox.rb
class InboundsMailbox < ApplicationMailbox
  before_processing :require_user

  def process
    extracted_urls = InboundEmailParser.new(mail.body.decoded).urls
    resource_title = mail.subject || "New resource"

    extracted_urls.each_with_index do |url|
      user.resources.create!(
        name: resource_title,
        url: url,
        source: "email"
      )
    end
  end

  private

  def require_user
    bounced! unless user
  end

  def user
    @user ||= InboundEmailAddress.find_by(email_address: mail.to)&.user
  end
end
```

A few intersting notes on the mailbox code:
- We use `bounced!` to stop processing the email if a corresponding user for the inbound email address does not exist
- The mail share extension will populate the mail subject with the resource page title, which is pretty handy !

The `InboundEmailParser.new` is also fairly simple, thanks to Ruby’s  `URI` module and its `extract` method (we only store `https` resources on Laterbox, hence the scheme validation):

```ruby
class InboundEmailParser
  def initialize(*body*)
    @body = *body*
  end

  def urls
    extract_urls
  end

  private

  def extract_urls
    URI.extract(@body, [ "https" ])
  end
end
```

And that’s it! Now resources can be saved natively from any mobile photo with email capabilities.   This code took a couple of hours of work for implementation, testing and release. Not bad for a feature that adds native capabilities to a web app and big improvement in user experience! Once again, ActionMailbox proved its value by offering a simple and famaliar controller-like interface to inbound emails.
