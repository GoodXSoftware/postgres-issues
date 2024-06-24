# Wrong security context in deferred triggers

Here is Laurenz reporting the issue to pgsql-hackers:
https://www.postgresql.org/message-id/flat/77ee784cf248e842f74588418f55c2931e47bd78.camel%40cybertec.at

The example in the thread is sufficient to demonstrate our problem. 

A [patch has been submitted for the July commitfest][commitfest].
There has been some interest and review, which is a good thing.
The reviewers ask for a good reason for this behaviour change, and the best I
can offer is &ldquo;it feels wrong the way it is&rdquo;.

Among the good criticism offered, one was: if we prevent the current user from
being changed between the triggering statement and the trigger execution, then
we should also prevent parameters like `search_path` being changed.  Now that
would be difficult, and I doubt that such an invasive change has a chance, but
the point is good.

I am kind of worried that the outcome will be: we should forbid the execution
of such triggers at all.  That would be a clean solution, but not one that will
make you happy.  So I mentioned that we have customers that have a business need
for the feature.

See [here][thread] for the e-mail thread and [here][email] for the e-mail I
am talking about.  I need some input from you.  Ideally, you would reply to
the message yourself and explain your use case.  To keep the thread (which is
important), you could either hit the [download mbox][download] link in the
e-mail and import the thread in your e-mail client, or you could hit the
&ldquo;resend email&rdquo; link.  You need to be subscribed to pgsql-hackers
for the latter, but you need to be subscribed anyway if you want to send a
reply.

If that is too complicated, you can of course also [write your response to
me][email] and I can relay it to the list, but it would lend more weight to
your request if you would chime in personally.  That would be one more voice
that wants the change.

 [commitfest]: https://commitfest.postgresql.org/48/4888/
 [thread]: https://www.postgresql.org/message-id/flat/77ee784cf248e842f74588418f55c2931e47bd78.camel@cybertec.at
 [email]: https://postgr.es/m/CAAvxfHceuGr0Cuc_mrpbH16a3dnsVA4QeOJ%2BkScvWbDiEo%2BU4Q%40mail.gmail.com
 [download]: https://www.postgresql.org/message-id/mbox/CAAvxfHceuGr0Cuc_mrpbH16a3dnsVA4QeOJ%2BkScvWbDiEo%2BU4Q%40mail.gmail.com
 [email]: mailto:laurenz.albe@cybertec.at
