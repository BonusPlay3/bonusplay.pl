<template>
	<v-layout column text-xs-justify>
		<!-- Goal -->
		<v-flex>
			<h1 class="display-3 text-xs-center">Mail server</h1>In this blog post, I'm going to describe how I setup mail server on my server.
			I want to be able to send and receive emails from domains I own (e.g. test@bonusplay.pl). Which means multiple users on multiple domains.
			I advise you to read <a href="https://wiki.archlinux.org/index.php/Mail_server">this page on arch wiki</a>.
			It's short, but it will help you understand everything.
			<v-img :src="require('@/assets/blog/mail-server/00.webp')" contain max-height="30vh"/>
		</v-flex>

		<v-divider class="my-3"/>

		<!-- Postfix + Dovecot -->
		<v-flex>
			<h2 class="display-2 text-xs-center">Attempt #1, postfix + dovecot</h2>So, at start I went with the most simple postfix + dovecot setup.
			Tbh, I just followed
			<a
				href="https://www.linode.com/docs/email/postfix/email-with-postfix-dovecot-and-mariadb-on-centos-7/"
			>this</a>
			amazing guide. I got it working pretty quickly.
			Remember to send some emails to your friends and tell them to mark it "not spam", since your domain will most likely be marked as spam.
			<br>
			<br>
			<i>fast forward a month...</i>
			<br>
			<br>
			<div class="has-text-centered">
				<v-img :src="require('@/assets/blog/mail-server/01.webp')" contain max-height="10vh"/>
			</div>Sending emails works perfectly, but gmail fetches POP3 emails once per hour?! WTF?!
			Ok, so we need to change our strategy...
		</v-flex>

		<!-- idea -->
		<v-flex>
			<h3 class="display-1 text-xs-center">Idea</h3>The one who gave me the idea is
			<i>Mateusz Maciejewski</i>, so kudos to him. Sending emails works just fine, so we don't have to touch this part.
			We only need to change how we receive emails. What if we don't store emails, but auto-forward them to designated email address?
			It removes waiting time, since email will be instantly forwarded, instead of being received, stored and then checked by gmail servers.
			<br>If so, why do we even need dovecot? Turns out, we need it for authentication. Then why did I switch from dovecot to cyrus?
			Amount of config dovecot needs is incredible, also
			<a
				href="https://serverfault.com/a/137007"
			>this</a> SO answer helped me move.
		</v-flex>

		<v-divider class="my-3"/>

		<!-- postfix + cyrus -->
		<v-flex>
			<h2 class="display-2 text-xs-center">Attempt #2, postfix + cyrus</h2>This part took me so much time. Like WAAAAAAAAAAAAY too much time.
			That's what I get for using arch linux on server. Ok, here it goes.
			<!-- auto forwarding -->
			<h3 class="headline">auto forwarding</h3>
			This was done using guide above. Just alias one email to another. I'm also using plain simple
			<code>aliases</code>
			file in tandem (but that's not needed).
			<!-- auth -->
			<h3 class="headline">authentication methods</h3>
			There are a few methods to choose from:
			<ul>
				<li>shadow - nope, since I want to have multiple accounts</li>
				<li>PAM - maybe? (I might switch to this one)</li>
				<li>IMAP server - nope</li>
				<li>sasldb - this is what I chose</li>
				<li>SQL - maybe? (I might switch to this one)</li>
				<li>LDAP - nope</li>
			</ul>First of all, be sure that you have
			<code>saslauthd</code> configured (in my case in
			<code>/etc/conf.d/saslauthd</code>) to actually use
			<code>sasldb</code>.
			You want the runtime argument to be
			<code>-a sasldb</code>.
			<br>Next is main config file (in my case
			<code>/etc/sasl2/smtpd.conf</code>).
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code0 }}</code></pre>Now, we need to add our accounts.
			<code>saslpasswd2 -c -u {domain} {username}</code> will do just that.
			<br>Test your accounts with
			<code>testsaslauthd -r {domain} -u {username} -p {password} -s smtp</code>
			.
			<!-- auth postfix -->
			<h3 class="headline">postfix config</h3>Now we just need to connect postfix to cyrus. Add this to
			<code>/etc/postfix/main.cf</code>.
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code1 }}</code></pre>
		</v-flex>

		<v-divider class="my-3"/>

		<!-- permissions -->
		<v-flex>
			<h2 class="display-2 text-xs-center">Permissions</h2>I run
			<code>saslauthd</code> service as
			<code>cyrus</code>cyrus user.
			Because postfix needs to communicate with cyrus over socket, I added both,
			<code>postfix</code> and
			<code>cyrus</code> accounts to
			<code>sasl</code> group.
			Be sure that group
			<code>sasl</code> has rw access to
			<code>/run/saslauthd</code> folder and
			<code>/etc/sasldb2</code>.
			Don't forget to set user in the
			<code>saslauthd</code> service file as well!
			<br>
			<br>
			<i>fast forward a month...</i>
			<br>
			<br>
			What do you mean, my emails are going to spam?! The answer for that is pretty simple.
			SMTP protocol has no security whatsoever. Which means anyone could send an email from
			<code>bill_gates@microsoft.com</code>. That's why everyne is sending our emails to spam.
			We need to somehow <b>prove</b>, that we really own the domain and we really are the real
			sender. To do that we need to add some stuff on DNS.
		</v-flex>

		<v-divider class="my-3"/>

		<!-- DMARC, etc. -->
		<v-flex>
			<h2 class="display-2 text-xs-center">Attempt #3, postfix + cyrus + opendkim</h2>
			First, we need to configure our DNS.
			<h3 class="headline">SPF</h3>
			Just use <a href="https://www.spfwizard.net/">this</a> record generator.
			<br/>
			Add the record as <code>TXT</code> on root level domain.
			<h3 class="headline">DMARC</h3>
			Pretty much same as <code>SPF</code>. Record generator <a href="https://mxtoolbox.com/DMARCRecordGenerator.aspx">here</a>.
			<br/>
			Add <code>TXT</code> record with <code>_dmarc</code> name.
			<h3 class="headline">DKIM</h3>
			Here's where the fun begins.
			<br/>
			<code>/etc/opendkim/opendkim.conf</code>
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code4 }}</code></pre>
			<code>/etc/opendkim/key.table</code>
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code5 }}</code></pre>
			<code>/etc/opendkim/signing.table</code>
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code6 }}</code></pre>
			<code>/etc/opendkim/trusted.hosts</code><br/>
			In this file add hosts that you trust (localhost, etc).
			Now, that we have <code>opendkim</code> set up, we need to generate our keys.
			<code>opendkim-genkey -b 4096 -h rsa-sha256 -r -d {domain} -v</code> will generate keys.
			Put private key in <code>/etc/opendkim/private</code> folder. Public key goes onto the DNS
			as <code>TXT</code> record with <code>mail._domainkey</code>.
			<h3 class="headline">Postfix</h3>
			Last thing to do is tell postfix to use opendkim (don't forget to fire up <code>opendkim</code> service after).
			<br/>
			Add in <code>/etc/postfix/main.cf</code>
			<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code7 }}</code></pre>
			There we go, now our emails are signed!
			<v-img :src="require('@/assets/blog/mail-server/05.webp')" contain max-height="30vh"/>
		</v-flex>

		<v-divider class="my-3"/>

		<!-- finished config -->
		<v-flex>
			<h2 class="display-2 text-xs-center">Done!</h2>
			Really? You might ask, wtf took me so long. I'm not the best sysadmin. Here is entire list of stuff I had problems with:
			<ul>
				<li>
					<code>saslauthd</code> is configured to use PAM by default (
					<code>-a pam</code>) on arch linux
				</li>
				<li>
					there is
					<code>-r</code> param in
					<code>saslauthd</code> which confused me
				</li>
				<li>
					<code>testsaslauthd</code> needed
					<code>-s smtp</code> and
					<code>-r {domain}</code> params
				</li>
				<li>opendkim couldn't create it's own runtime folder</li>
				<li>many many more, that I erased from my memory...</li>
			</ul>My hints for debugging are:
			<ul>
				<li>
					check
					<code>/var/log/auth.log</code> and
					<code>/var/log/mail.log</code>
				</li>
				<li>
					use
					<a href="https://wiki.zimbra.com/wiki/Simple_Troubleshooting_For_SMTP_Via_Telnet_And_Openssl">this</a> to check via telnet (often gives some output)
				</li>
			</ul>
		</v-flex>

		<!-- full config -->
		<v-flex>
			<v-expansion-panels>
				<v-expansion-panel>
					<v-expansion-panel-header>
						<span>
							<h4 class="title is-4">Bonus</h4>
							Full postfix configuration
						</span>
					</v-expansion-panel-header>
					<v-expansion-panel-content>
						<code>/etc/postfix/main.cf</code>
						<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code2 }}</code></pre>
						and <code>/etc/postfix/master.cf</code>
						<pre class="line-numbers" data-start="0"><code class="lang-none">{{ code3 }}</code></pre>
					</v-expansion-panel-content>
				</v-expansion-panel>
			</v-expansion-panels>
		</v-flex>

		<v-divider class="my-3"/>

		<!-- bonus (gmail) -->
		<v-flex>
			<v-expansion-panels>
				<v-expansion-panel>
					<v-expansion-panel-header>
						<span>
							<h4 class="title is-4">Bonus #2</h4>
							Did you know gmail has dark theme?
						</span>
					</v-expansion-panel-header>
					<v-expansion-panel-content>
						<v-img :src="require('@/assets/blog/mail-server/02.webp')" contain />
						Here's how to set it:
						<v-img :src="require('@/assets/blog/mail-server/03.webp')" contain/>
						There are also configurable filters. Here is an example filter I use, to label all emails that come from my website.
						<v-img :src="require('@/assets/blog/mail-server/04.webp')" contain />
					</v-expansion-panel-content>
				</v-expansion-panel>
			</v-expansion-panels>
		</v-flex>

		<v-divider class="my-3"/>

		<!-- done -->
		<v-flex>
			<h1 class="display-2 text-xs-center">Done!</h1>
			Nothing more. Enjoy your instant, free emailing!
		</v-flex>
	</v-layout>
</template>

<script>
// import Prism from "prismjs";

export default {
	name: "MailServer",
	mounted() {
		// Prism.highlightAll();
	},
	data: () => ({
		code0: require('@/assets/blog/mail-server/00.txt').default,
		code1: require('@/assets/blog/mail-server/01.txt').default,
		code2: require('@/assets/blog/mail-server/02.txt').default,
		code3: require('@/assets/blog/mail-server/03.txt').default,
		code4: require('@/assets/blog/mail-server/04.txt').default,
		code5: require('@/assets/blog/mail-server/05.txt').default,
		code6: require('@/assets/blog/mail-server/06.txt').default,
		code7: require('@/assets/blog/mail-server/07.txt').default
	})
}
</script>