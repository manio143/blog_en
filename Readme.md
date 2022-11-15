# Marian's blog

https://devblog.dziubiak.pl/

Hi there! I'm Marian, a software engineer, big fan of .NET and this my blog for writing down my coding adventures.
I used to write a blog in Polish over on <https://www.md-techblog.net.pl> but lately I've been thinking I need a new outlet in English.

## Generating images
Install [imagemagick](https://imagemagick.org/script/download.php) and run

```
magick .\assets\post_image_template.png -font Consolas -pointsize 62 -fill white -annotate +450+330 "<title>" img.jpg
```

You may need to adjust font size and positioning to have it fit. For more info on parameters see [documentation](https://imagemagick.org/Usage/text/#annotate)
