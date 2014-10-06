---
layout:     post
title:      "Foodspotting for Android: Improving the Map Experience"
date:       2013-08-27 11:53:19
summary:    A study of techniques used to improve the map experience in Foodspotting for Android using marker clustering, custom overlays, and animation.
categories: android foodspotting
---

## Introduction ##

[Foodspotting](http://www.foodspotting.com) from [OpenTable](http://www.opentable.com) is an app for finding and recommending great dishes, not just restaurants. With this visual guide to good food and where to find it, you can find whatever you’re craving, see what’s good at any restaurant and learn what foodspotters, friends and experts love wherever you go.

Foodspotting dish photos naturally collect around discrete places - restaurants, food carts, stands, and stalls. If we were to put a marker on the map for every one of them, many of them would stack on top of each other with no indication that there are multiple dishes nor what the individual dishes are.

## Clustering ##

One way we can indicate that there are multiple items at some map locations is by using two kinds of markers: one for single items and another for stacks of items. We just need a way to organize our data into co-located clusters.

**Marker clustering implementation**
_There are lots of ways to find the clusters in your data. We kept it simple (and not as accurate as possible). Wikipedia has more sophisticated techniques at [http://en.wikipedia.org/wiki/Cluster_analysis](http://en.wikipedia.org/wiki/Cluster_analysis)._

Since we want to find items that will be displayed on top of each other on the screen, we project them from latitude/longitude space into screen space then put them into the nearest cluster:

{% highlight java %}
for (Item item : items) {
   // Project lat/lng into screen space, given map's current zoom, etc.
   Point p = map.getProjection().toScreenLocation(item.getPosition());
   // Find the first cluster near point p
   Cluster cluster = null;
   for (Cluster c : clusters) {
      if (c.contains(p)) {
         cluster = c;
         break;
      }
   }
   // Create a new Cluster if there were none nearby
   if (cluster == null) {
      cluster = new Cluster(p);
      clusters.add(cluster);
   }
   cluster.add(item);
}
{% endhighlight %}

The test for whether a `Point` is in a Cluster could be as simple as, "is it within a `Rect`?":

{% highlight java %}
class Cluster {
   Rect bounds;
   public Cluster(Point p) {
      // Delta should be in DIP units - so scale it by screen density!
      int delta = getClusterTolerance();
      bounds = new Rect(p.x - delta, p.y - delta, p.x + delta, p.y + delta);
   }
   public boolean contains(Point p) {
      return bounds.contains(p.x, p.y);
   }
   // ... addItem(), getNumItems(), etc.
}
{% endhighlight %}

Then all we need to do is create a map `Marker` for each cluster:

{% highlight java %}
for (Cluster cluster : clusters) {
   int numItems = cluster.getNumItems();
   Item item = cluster.getItem(0);
   int resId = (numItems > 1 ? R.drawable.marker_multi : R.drawable.marker_single);
   MarkerOptions opts = new MarkerOptions()
      .position(item.getPosition())
      .icon(BitmapDescriptorFactory.fromResource(resId));
   Marker marker = map.addMarker(opts);
}
{% endhighlight %}

The last consideration for clustering is that since clusters are defined by screen-space distance, we need to rebuild them when the zoom level changes. _Hint: Use a `GoogleMap.OnCameraChangeListener` and track the `CameraPosition.zoom` values._

## Peeking inside the cluster ##

Maps are great for context and navigation but content is king, as they say. We want users to focus on the content not the map. How do we do a good job of uncovering the content available at a cluster?

## Clusters are lists ##

The new Maps API led us to rethink our design. In the previous design, we allowed the user to tap cluster markers multiple times to cycle through the elements in the cluster. That's hidden functionality and not very rich visually.

Our insight seems simple in retrospect: We can use a `ListView` to display an unlimited number of elements in a cluster, in a limited amount of space. And we can use `ListView` to repeat UI design elements from elsewhere in the app to keep things cohesive. And it looks like a visual restaurant menu! Plus, with this design, if all the dishes are from the same place, we can give the user a clear way to get more information about that place.

That leaves us with the challenge of how to arrange the screen so that we have enough room to display the list and indicate which cluster is being examined. If we also reposition the map around the content so that the focused cluster is always at the top (or left) then we can just overlay the `ListView` on the window and we don't need any tricky positioning or sizing logic. 

**Info window implementation**

Here's the sketch of the idea:

* The info window is a full screen `FrameLayout` with a `ListView` where the `topMargin` (in portrait, `leftMargin` in landscape) is set to accommodate the `Marker`.
* Move the map so that the `Marker` is at the top (or the left) of the screen leaving lots of room to display the list.
* Add a list footer button to see more about the place if every item is from the same place.
* Make it work in landscape, too. `s/top/left/g`
* Put things back where they were when we're done.

![Info window](https://googledrive.com/host/0B99yjQUorG9jQ3NVclRCelpIN0U/info-overlay-explode.png)

The info window layout needs to cover the whole screen but allow us to position the list inside of it:

{% highlight xml %}
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
   android:id="@+id/fullscreen_overlay"
   android:layout_width="match_parent"
   android:layout_height="match_parent" >
   <!--
    We need this inner FrameLayout because we'll position
    it within its parent using top and left margins 
   -->
   <FrameLayout
      android:id="@+id/info_window"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:layout_gravity="top|left"
      android:background="@drawable/info_window_bg_up" >
      <ListView
         android:id="@android:id/list"
         android:layout_width="match_parent"
         android:layout_height="wrap_content" />
   </FrameLayout>
</FrameLayout>
{% endhighlight %}

Now we can figure out the where the marker will be once we reposition the map, show the info window at the marker, and move the map camera appropriately:

{% highlight java %}
// Get the quadrant of the screen where the marker will be
// top zone: Upper 1/4 of screen, left zone: Leftmost 1/4 of screen
int cx = w / 2; // horizontal center of screen
int cy = (h / 4) / 2; // vertical center of quadrant
// if landscape: cx = (w / 4) / 2; cy = h / 2;
Point newMarkerPos = new Point(cx, cy);

// Get the current marker position
Marker marker = cluster.getMarker();
// Get the screen-space position of the Cluster
Point p = map.getProjection().toScreenLocation(marker.getPosition());
int markerX = p.x;
int yoffset = markerSize / 2; // vertical center of marker
int markerY = p.y - yoffset;
      
// Position the selected marker in the top (or left) zone by moving the camera
map.animateCamera(calculateCameraChange(newMarkerPos.x, newMarkerPos.y, markerX, markerY),
      400, null);
// Show the cluster list next to the marker, padding x and y a little
int xoffset = markerSize / 2;
showInfo(newMarkerPos.x + xoffset, newMarkerPos.y + yoffset, cluster);
{% endhighlight %}

We can calculate the necessary camera change by projecting the screen space change into latitude/longitude space:

{% highlight java %}
private CameraUpdate calculateCameraChange(int newX, int newY, int oldX, int oldY) {
   // WARNING: This is broken when the map is tilted!
   Projection proj = map.getProjection();
    // Calculate the change in position _in screen space_
   Point cameraPos = proj.toScreenLocation(map.getCameraPosition().target);
   int dx = newX - oldX;
   int dy = newY - oldY;
   cameraPos.x -= dx;
   cameraPos.y -= dy;
    // Convert the screen space change into lat/lng space
   return CameraUpdateFactory.newLatLng(proj.fromScreenLocation(cameraPos));
}
{% endhighlight %}

And then position the list next to the marker:

{% highlight java %}
private void showInfo(int x, int y, final Cluster c) {
   int orientation = getResources().getConfiguration().orientation;   
   // Populate the ListView with the Cluster's Items 
   // …
   int defaultMargin = getDefaultMargin(); // 16dip would be good here
   
   // Reconfigure the layout params to position the info window on screen
   FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) infoWindow.getLayoutParams();
      
   if (orientation == Configuration.ORIENTATION_PORTRAIT) {
      lp.topMargin   = y;
      lp.leftMargin  = defaultMargin;
      lp.rightMargin = defaultMargin;
      lp.width       = LayoutParams.MATCH_PARENT;
      lp.height      = LayoutParams.WRAP_CONTENT;
      lp.gravity     = Gravity.LEFT | Gravity.TOP;
      infoWindow.setBackgroundResource(R.drawable.info_window_bg_up);
   } else if (orientation == Configuration.ORIENTATION_LANDSCAPE) {
      lp.leftMargin   = x + defaultMargin;
      lp.topMargin    = defaultMargin;
      lp.bottomMargin = defaultMargin;
      lp.gravity      = Gravity.LEFT | Gravity.CENTER_VERTICAL;
      lp.width        = LayoutParams.WRAP_CONTENT;
      lp.height       = LayoutParams.WRAP_CONTENT;
      infoWindow.setBackgroundResource(R.drawable.info_window_bg_left);
   }
   infoWindow.setLayoutParams(lp);
   fullScreenOverlay.setVisibility(View.VISIBLE);
}
{% endhighlight %}

_Don’t forget to put the map back where it was when the user is done with the info window!_ Before you move the camera, store its current position. When the user dismisses the popup list (probably by tapping outside of it), restore the previous camera position.

Here's how Foodspotting's implementation looks in portrait and landscape modes:

Portrait | Landscape
-------- | ---------
![Info Window Portrait](https://googledrive.com/host/0B99yjQUorG9jQ3NVclRCelpIN0U/info-window-port-thumb.png) | ![Info Window Landscape](https://googledrive.com/host/0B99yjQUorG9jQ3NVclRCelpIN0U/info-window-land-thumb.png)



## Spotlight ##

### Turn the lights down low ###

To really bring the focus to the content, we can dim the map and animate the change from “browse mode” to "info window mode" by making a spotlight effect animation.

![Spotlight Animation](https://googledrive.com/host/0B99yjQUorG9jQ3NVclRCelpIN0U/spotlight-animation.gif "Spotlight Animation")


**Spotlight implementation**

Here's the basic idea:

* The spotlight is just a radial `GradientDrawable` on a full-screen `FrameLayout`.
* Project marker position into screen space using the current map projection.
* Gradient center coordinates are specified in the range [0,1] where 1 corresponds to the full width or height of the drawable.
* Scale marker x coordinate from [0,screen_width) → [0,1) and y coordinate from [0,screen_height) → [0,1) to use as the radial gradient center coordinates.
* Animate old gradient center to new gradient center.

{% highlight java %}
// Show the spotlight where the marker currently is by computing the
// screen position as a fraction of the whole area
float cx = markerX / (float)getWidth();
float cy = markerY / (float)getHeight();
setSpotlight(cx, cy);

// Animate the spotlight from old to new position
SpotlightAnimation spotAnim = new SpotlightAnimation(markerX, markerY, newMarkerPos.x, newMarkerPos.y);
this.startAnimation(spotAnim);
{% endhighlight %}

where we set the spotlight gradient's center directly:

{% highlight java %}
private void setSpotlight(float cx, float cy) {
   spotDrawable.setGradientCenter(cx, cy);
   // The following line is needed for Android 2.2 to flag it's state
   // as dirty and pick up the new gradient center set above.
   spotDrawable.setGradientType(GradientDrawable.RADIAL_GRADIENT);
}
{% endhighlight %}

and wrap up our animation in a nice subclass:

{% highlight java %}
public class SpotlightAnimation extends Animation {
   float cx0, cy0, dx, dy;
   public SpotlightAnimation(final int startX, final int startY,
         final int endX, final int endY) {
      set(startX, startY, endX, endY);
   }
   public void set(final int startX, final int startY, final int endX, final int endY) {
      final float w = getWidth();
      final float h = getHeight();
      cx0 = (float) startX / w;
      cy0 = (float) startY / h;
      dx = (endX / w) - cx0;
      dy = (endY / h) - cy0;
   }
   @Override
   protected void applyTransformation(final float interpTime, final Transformation t) {
      final float cx = cx0 + interpTime * dx;
      final float cy = cy0 + interpTime * dy;
      setSpotlight(cx, cy);
   }
   @Override
   public boolean willChangeTransformationMatrix() { return false;  }
   @Override
   public boolean willChangeBounds() { return false; }
}
{% endhighlight %}

The keen-eyed among you will have noticed that there is no synchronization between the spotlight animation and the camera animation. There is currently no way to guarantee synchronization. Even camera position change events are not guaranteed to fire at any specific time. One approach to deal with that problem is to fade in the spotlight as it animates (and cross your fingers!)


## Conclusion ##

Use maps for what they're good for, then focus on the content. Try to find the simplest ideas. Dress in layers.

## Get the code ##

Get the sample code demonstrating all of these ideas from Github at:

[https://github.com/opentable/google-maps-blog-android-sample-code](https://github.com/opentable/google-maps-blog-android-sample-code)

Get the free OpenTable and Foodspotting apps from the Google Play store at:

![Google Play Store][1]{: .inline}  [OpenTable - Free Reservations](https://play.google.com/store/apps/details?id=com.opentable)

![Google Play Store][1]{: .inline}  [Foodspotting](https://play.google.com/store/apps/details?id=com.foodspotting)

[1]: https://s2.googleusercontent.com/s2/favicons?domain=play.google.com&alt=p

---

_[OpenTable](http://www.opentable.com) is the world's leading provider of online restaurant reservations, seating more than 12 million diners per month via online bookings across approximately 28,000 restaurants. The OpenTable network connects restaurants and diners, helping diners discover and book the perfect table and helping restaurants deliver personalized hospitality to keep guests coming back. The OpenTable service enables diners to see which restaurants have available tables, select a restaurant based on verified diner reviews, menus and other helpful information, and easily book a reservation._

_[Foodspotting](http://www.foodspotting.com) is an app for finding and recommending great dishes, not just restaurants. With this visual guide to good food and where to find it, you can find whatever you’re craving, see what’s good at any restaurant and learn what foodspotters, friends and experts love wherever you go. Foodspotting was acquired by OpenTable in 2013 who has continued to support it as a standalone app._
 
