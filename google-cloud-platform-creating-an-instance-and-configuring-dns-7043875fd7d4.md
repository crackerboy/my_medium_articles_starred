
# Google Cloud Platform: Creating an Instance and Configuring DNS



For the last few weeks I’ve been enamored with Google’s cloud platform, aptly named Google Cloud Platform. GCP contains the things you might expect from a young player in the ‘screw AWS’ space: much of what exists on AWS has an equivalent on GPC, but certain subtleties exist, such as the lack of Python serverless functions and so forth. That said, GCP makes up for any shortcomings by leveraging services exclusive to Google.

In my opinion, GCP is the first contender in the market to package enterprise cloud computing in a satisfying way. Its clear GCP has assigned UI and Product Management resources to their platform, where Amazon clearly did not. while not without its shortcomings, it’s obvious Google has chosen usability as a key differentiator from AWS.

Aside from the UI, GCP offers plenty of fun functionality such as their cloud launcher. This is the equivalent of one-click deploys for cool stuff, whether they be services to add to your VPC, Google APIs, Datasets, or what have you. If you’re the type of person who browses Product Hunt with a mild SaaS addiction, good luck escaping from Cloud Launcher in a reasonable amount of time.

![It’s like Product Hunt… on crack.](https://cdn-images-1.medium.com/max/2292/0*UxWLRPRNPkxqjFpr.gif)*It’s like Product Hunt… on crack.*

The ease of plug-and-play these plug-and-play services make GCP a compelling choice for a respectable enterprise which hasn’t lost the gift of curiosity just yet. Let’s set up a server to see what the hype is about. Today’ll we’ll be creating a standard Linux instance and walking through the steps to configure an existing DNS for said instance.

## Does in Fact Compute

GCP cutely names their server’s *Compute Engines,* which are at least more tolerable than, say, EC2 instances. I’m just going to call them servers because I’m not the type of person who orders a “tall” at Starbucks.

Create a “project” in Google Cloud, and you’ll immediately land at a dashboard. All Google’s services are tucked away in the left-hand menu. Open that bad boy up and find Compute Engine.

![Shhh, it’s thinking.](https://cdn-images-1.medium.com/max/2640/1*UcKGdjHlmlLtHSCF0SonZg.gif)*Shhh, it’s thinking.*

Select *create*. As opposed to the preset choices of VPCS you might be used to, Google allows us to customize our VPS to our exact technical specifications on a sliding scale. Want 96 processing cores, but only a single GB of RAM? No problem, if that’s what y0u’re into. Weirdo.

As well as picking between the usual Linux distros, Compute Engine also allows customers to select their number of GPUs, as well as the generation of Intel CPU their instance will run on.

![Dat customization tho.](https://cdn-images-1.medium.com/max/3416/0*jVMDKSdIx-zogP23.png)*Dat customization tho.*

We want traffic to hit this instance, so make sure you check **Allow HTTP** **traffic** and **Allow HTTPS traffic** before continuing. Once your instance is created, you should immediately able to SSH into your server via GCP’s browser client.

## The App Engine

GCP is not without its own fair share of arbitrary product classifications. DNS records and hosts are contained within the **App Engine** service of the platform. Find the App Engine service in the left hand navigation, and scroll down to the *settings* link:

![Allllll the way at the bottom.](https://cdn-images-1.medium.com/max/3712/0*RDCFkNqArF5LfZ_k.png)*Allllll the way at the bottom.*

Here’s we’ll be able to see a “custom domains” tab which allows us to point a domain we’ve purchased from a service like *Namecheap *or what-have-you to Google Cloud. I’ll personally be walking though this example by directing a pointless domain called *memegenerator.io* I purchased on Namecheap for no good reason.

When you add a custom domain via this screen, you’ll immediately be asked to verify ownership over the domain via the familiar Google Webmaster tool, which you’ll be redirected to automatically.

## Back to Your Registrar

Chances are you’ve dealt with verification via Google webmaster before, but this time we’ve only given the option to do this via DNS. Select your registrar in the dropdown in order to reveal a Google-generated record used to verify your domain.

![Please don’t tell me you use GoDaddy.](https://cdn-images-1.medium.com/max/3864/0*QuKb8DD3WgCy4Rvd.png)*Please don’t tell me you use GoDaddy.*

The resulting value will need to be added as a .txt record before we can actually point your domain to Google’s servers.

If you’re using Namecheap like I am, log in to your dashboard and find your domain by clicking “manage domain”. Make sure you’re under the “advanced DNS” tab:

![Even if you’re not using Namecheap, this shouldn’t be much different.](https://cdn-images-1.medium.com/max/3368/1*dKYWuVZn7CvVgOuUpiAOVg.png)*Even if you’re not using Namecheap, this shouldn’t be much different.*

Delete all existing records. Then create a TXT record (with @ as the host) and input the value that Google provided you earlier. Now, when you return to the webmaster tool, clicking verify tool *should* pick up on this change.
> If the webmaster tool does not pick up on this verification right away, don’t panic. This happens fairly often — just frantically keep making sure everything is set up correctly while mashing the verify button.

Navigate back to the Custom Domains tab in GCP and continue the process- you should see that your domain is verified. You’ll be prompted to enter any subdomains you’d GCP to pick up on here. Wrap that up and save.

![](https://cdn-images-1.medium.com/max/2064/0*Wrjrq56eg5D1tp2Q.png)

## Make it Rain with Records

Oh, we’re far from over buddy. We need to go back to update our A and AAAA records, now that Google as bestowed that privilege upon us. You should see a table such as the one below:

    Type     Data                    Alias

    A        216.239.32.21
    A        216.239.34.21 
    A        216.239.36.21 
    A        216.239.38.21 
    AAAA     2001:4860:4802:32::15 
    AAAA     2001:4860:4802:34::15 
    AAAA     2001:4860:4802:36::15 
    AAAA     2001:4860:4802:38::15 
    CNAME    ghs.googlehosted.com     www

Copy that into your registrar’s custom DNS records. Have fun with that.

## Cloud DNS

You may have noticed that we haven’t actually specified our Nameservers yet. Nobody said this was going to be fun; if it were, we probably wouldn’t need this tutorial. In the GCP search bar, search for *Cloud DNS*. Create a Zone, and leave DNSSEC off.

![](https://cdn-images-1.medium.com/max/2076/0*zU_lbiW1oWFiLXUC.png)

Before we do this next part, I’d like to interject and mention that you did a spectacular job of creating all those records and pasting all those values earlier. Considering you seem to have a knack for this, it probably won’t hurt to know that we need to go back into our registrar a third time to paste some values. You got this.

Google’s nameservers have now been generated and exposed to you so we can *actually* point our domain to something meaningful now. You should have 4 nameservers like the following:

    Name                Type      TTL   Data 
    memegenerator.io.    NS     21600   ns-cloud-b1.googledomains.com. 
                                        ns-cloud-b2.googledomains.com.
                                        ns-cloud-b3.googledomains.com.
                                        ns-cloud-b4.googledomains.com.

## Assign a Static IP

Okay, we’re officially done messing around with our registrar. In the GCP search bar, search for **External IP addresses.** From there click the “Reserve static Address” button at the top of the screen. This will prompt you with a short form: the only important field to fill out here is the *“Attached to”* dropdown, which denotes which server instance the IP will be assigned to.

## Compute Engine Instance Settings

Shit, is there even more? OK, we’re almost done here. Go to your Compute Engine instance you set up earlier. Click “Edit”. Scroll to the *Network Interface* section and map the Static IP we created from earlier. Also, go ahead and enter your PTR record:

![](https://cdn-images-1.medium.com/max/3056/0*ttJSisKEHK_zH0xy.png)

## FINAL CHAPTER: Firewall Settings

Look, I just want to say you’re all doing a great job so far. All of you. We’re all a team here; let’s stick together and see this through. Search for **Firewall Rules** and selected *Create a Firewall Rule. *Name it whatever you want.

* **Targets** — This will be where our traffic routes. We want to route to our instance, which is a *specified service account.*

* **Target service account** — Referring to the above, this is where we select the computer instance we want to hit.

* **Target service account** **scope** — Select “in this project”.

* **Source Filter **— Once again, select *specified service account.*

* **Source service account** **scope **— Select “in this project”

* **Source service account** — This is where we say where the traffic is coming from. It’s coming from the *App engine*, as this is where we specified our DNS.

* For **IPs** and **ports, **well, do what you want.

## Get at it

Well, there you have it. Hopefully by now the domain you’ve painstaking configured now points to your server, so you can go ahead and configure your webserver settings or whatever it is you do.

Alright fine, so GCP isn’t completely free of its own redundancies. As much as I love to hate on AWS, it seems almost inevitable at this point that any enterprise cloud service will maintain a certain level of obscure processes. This is great for flexibility when scaling, but let’s be honest: if these platforms were easy to use, who would pay to become certified in using them?

*Originally published at [hackersandslackers.com](https://hackersandslackers.com/setting-up-dns-with-google-cloud-platform/) on July 14, 2018.*