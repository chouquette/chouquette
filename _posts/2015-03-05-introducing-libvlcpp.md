---
published: false
---

## Introducing: libvlcpp

You probably already know VLC. As you might know, and as j-b mentionned in his [talk during FOSDEM](http://video.fosdem.org/2015/devroom-open_media/vlc.mp4), «VLC doesn't exist».
Indeed, VLC is basically a 100 lines wrapper around libvlc. This library is the entry point to another one, libvlccore, which is doing all the heavy lifting, coordinates modules with each other.

libvlc is a public API, and can be used by anyone wishing to add multimedia functionnalities to its application.
While it's a fairly easy to use API, it contains quite a few things which can be annoying to C++ developers. For instance, non automatic ref-counting, callbacks with raw function pointers, raw C strings, ... 

In the last few years, VideoLAN had quite a bunch of C++ wrappers. I can count 4 from the top of my head, and a quick search on Google gives out a few more. All have their specificities, advantages, and drawbacks.
Since most of them are not maintained nor they expose the full set of functionalities available within libvlc, j-b and I decided to write one, that will, hopefully, be the "official" one.

It just has been tagged as 0.1.0-rc1, and is now waiting for applications to use it!

Before showing a quick sample, here's a quick summary of libvlcpp:
- Automatic VLC objects ref-counting
- Handlers events with safe lambdas
- Header only implementation
- Full VLC 2.2 functionnalities set.
- C++11

Now for a quick example, which is included in the git as the "helloworld" project:

{% highlight c++ %}
#include "vlcpp/vlc.hpp"
#include <thread>

int main(int ac, char** av)
{
    if (ac < 2)
    {
        std::cerr << "usage: " << av[0] << " <file to play>" << std::endl;
        return 1;
    }
    auto instance = VLC::Instance(0, nullptr);
    auto media = VLC::Media(instance, av[1], VLC::Media::FromPath);
    auto mp = VLC::MediaPlayer(media);
    mp.play();
    std::this_thread::sleep_for( std::chrono::seconds( 10 ) );
}
{% endhighlight %}

Which is only going to create a new window, and display the video you provided in it.
There's obviously ways to make things look better, but that will be part of some other example projects :)

As said earlier, this has been tagged as a rc1, so we need some your reviews & comments!
I hope you'll enjoy using libvlcpp as I enjoyed writing it!