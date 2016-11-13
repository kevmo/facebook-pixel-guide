# A guide to using the Facebook Pixel

Facebook allows us many different ways to target users with advertising. Facebook rewards advertisers for posting relevant ads. Not only money can be saved by targeting people who are interested in the product being advertised, of course, but also [Facebook will charge less](https://vupload.facebook.com/business/help/community/question/?id=280840818962760) per impression if there are positive interactions between the user and the ad such as a click through, like, or share.

There are lot of people angry about Facebook's use of personal information. Yes, but it is there and hopefully can be used in positive ways.

The Facebook Pixel embedded in a page passes a Facebook cookie to Facebook which is used to check if the user is also a Facebook user. Facebook will allow you to target advertisements to any person associated with that page view on your website. This is how I use it in the context of a single page AngularJS app.

Here is the link to the [Facebook Pixel Implementation Guide](https://www.facebook.com/business/help/952192354843755)

#####Contents

1. Motivation
+ Privacy statement
+ Embedded pixel code
+ Capturing page interactions
+ Creating a custom audience using a Pixel

## 1. Motivation
I built a job board for the yachting and marine industry, [Simple Yacht Jobs](http://www.simpleyachtjobs.com/), and I want both to promote it and to let people know if a job they might be interested in becomes available. The website knows if a person is interested in a type of job if they request more information for a particular job of that type already listed.

Following is how I translate that action into letting that person know -- if they are a Facebook user which in the case of yachting most people are because they are working in places like the Caribbean and it connects them to friends and family back home in place like New Zealand, South Africa, and France -- that a job has become available. Rather than seeing another ad for a cell phone on Facebook, I'm going to help them find a job.

## 2. Privacy statement
Letting people know what data you are collecting is not only a legal requirement but also the honest, decent thing to do. I include this statement in the privacy statement. 
 
 >Similar to other commercial websites, our website utilizes a standard technology called “cookies” (see explanation below, “What Are Cookies?”), Web server logs (“Log Files”), Google Analytics, and Facebook Pixel to collect information about how our website is used. Information gathered through cookies, Google Analytics, Facebook Pixel and Web server logs may include the date and time of visits, the pages viewed, information about job and resume profile listings you open and view, time spent at our website, and the websites visited just before and just after our website. We, our advertisers and ad serving companies may also use small pieces of code called “web beacons” or “clear gifs” to determine which advertisements and promotions users have seen and how users responded to them.
 
 >Personally identifiable information is not stored in these cookies, web beacons, or clear gifs. We use third-party tracking services, Google Analytics and Facebook Pixel, that use cookies and log files to track visitors to our site in the aggregate, usage and volume statistic to help us better understand the needs of our visitors. Although we do not send personally identifiable information, e.g. email address or phone number, to Google Analytics, we do send an obfuscated user id associated with your account if you are logged in with an active session per Google Analytics terms of service.
 
 >You can **opt out** of being tracked by Google Analytics, Facebook Pixel, and most third party tracking services by using an ad blocker plugin for your browser. For example, [AdBlock](https://getadblock.com/) is a very popular ad and tracker blocker which will prevent this site from tracking you with Google Analytics and Facebook Pixel.

## 3. Embedded Facebook Pixel code
I use Node.js and Express so I compile the `index.html` file with Swig on initial page load to include settings variables that get passed along from a config file and process variables to a global `window.values` object in the client. One of the settings values is `facebookPixelId` which contains my personal Pixel Id.
  
  You can find your Pixel Id in the [Facebook Ads Console](https://www.facebook.com/ads/manager/pixel/facebook_pixel).
  
  There are two custom events inside the embedded Pixel code which need to be subscribed, the `init` event which contains your Pixel Id as the second parameter and the `track` event which contains what you want to track which in this case is `PageView`.
  
  In a single page application, the `PageView` event detects when `window.location` changes. So if you are using a router, every time the location changes the `PageView` event is fired and a message is sent to the Facebook servers that the user is on a different page. If you just change the search query parameters, it also gets fired but throws a warning message for reasons.
   
  Here is the embedded code in the `index.html` file.
  
      <script>
          !function(f,b,e,v,n,t,s){if(f.fbq)return;n=f.fbq=function(){n.callMethod?
                  n.callMethod.apply(n,arguments):n.queue.push(arguments)};if(!f._fbq)f._fbq=n;
              n.push=n;n.loaded=!0;n.version='2.0';n.queue=[];t=b.createElement(e);t.async=!0;
              t.src=v;s=b.getElementsByTagName(e)[0];s.parentNode.insertBefore(t,s)}(window,
                  document,'script','//connect.facebook.net/en_US/fbevents.js');
          // Insert Your Facebook Pixel ID below.
          fbq('init', window.values.facebookPixelId);
          fbq('track', 'PageView');
      </script>
  
  I didn't bother with the static fallback `<img>` pixel code because the site is useless without JavaScript enabled.
  
## 4. Capturing page interactions
I created an AngularJS service that has for a `logEvent` method that makes a Facebook Pixel track call when called. It stores information about the event such the type of job a user was looking at and the position of the job, for example, yacht chef or captain. 
  
````
  (function() {
      angular.module('simplejobs')
          .factory('sjLogger', sjLogger);
  
      sjLogger.$inject = ['$window', 'Analytics', 'Auth'];
  
      function sjLogger($window, Analytics, Auth) {
          var googleAnalyticsEvent = Analytics.trackEvent,
              facebookPixelEvent = $window.fbq;
  
          return {
              logEvent: logEvent
          };
  
  
          /**
           * Google Analytics
           * ----------------
           * Category: Job || Resume
           * Event action: ViewListing || EditListing || DownloadResume
           * Dimension 1: JobType
           * Dimension 2: Position
           * Dimension 3: LocationName
           * Dimension 4: VesselType
           * Dimension 5: UserId
           *
           * Facebook Pixel
           * ________________
           * EventAction: ViewListing || EditListing || DownloadResume
           * Event (Category): Job || Resume
           * EventLabel: JobType
           * Position: Position
           * LocationName: LocationName
           * VesselType: VesselType
           *
           * event: {
           *     action: [string],
           *     category: [string],
           *     jobType: [string],
           *     position: [string],
           *     locationName: [string],
           *     vesselType: [string]
           * }
           */
          function logEvent(event) {
              var user = Auth.getMe();
  
              // (category, action, label, value, noninteraction, custom)
              // The event position value in the label is redundant but semantically incorrect in the analytics dashboard it should
              // be labeled position under a dimension but Event label is a major default dimension. So I'm sacrificing one of
              // the 100 custom dimensions for redundancy. I can change this later if it doesn't make sense. Deal with it.
              googleAnalyticsEvent(event.category, event.action, event.position, {
                  dimension1: event.jobType,
                  dimension2: event.position,
                  dimension3: event.locationName,
                  dimension4: event.vesselType,
                  dimension5: user._id
              });
  
              facebookPixelEvent('trackCustom', event.action, {
                  Category: event.category,
                  JobType: event.jobType,
                  Position: event.position,
                  LocationName: event.locationName,
                  VesselType: event.vesselType
              });
  
          }
      }
  })();
  ````
  
This method is called in the job listing information dialog controller is created when someone clicks the more information about the job button. 

````
(function() {
    angular.module('simplejobs')
        .controller('searchJobsDialogCtrl', searchJobsDialogCtrl);

    searchJobsDialogCtrl.$inject = ['$mdDialog', 'loginService', 'job', 'sjLogger'];

    function searchJobsDialogCtrl($mdDialog, loginService, job, sjLogger) {
        var dialog = this;

        dialog.job = job;
        dialog.login = loginService.open;

        sjLogger.logEvent({
            action: 'ViewListing',
            category: 'Job',
            jobType: job.jobType,
            position: job.position,
            locationName: job.location.name,
            vesselType: job.vesselType
        });

        dialog.close = function() {
            $mdDialog.hide();
        };

        dialog.cancel = function() {
            $mdDialog.cancel();
        };
    }
})();
````

Facebook will associated this information sent with the Facebook user.

## 5. Creating a custom audience using a Pixel

One the Pixel dashboard, click the Create Audience button to create a custom audience using data stored from the Pixel.