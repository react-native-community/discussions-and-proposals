# RFC: Faster capHeight-baseline rendering in &lt;Text&gt;

## Background

- [baseline](https://en.wikipedia.org/wiki/Baseline_(typography))
- [cap height](https://en.wikipedia.org/wiki/Cap_height)
- [article with some information about bounding box issues on mobile](https://www.prolificinteractive.com/2016/03/08/all-about-that-baseline-3/)

## Proposal

A few months ago, we added an onTextLayout property to &lt;Text&gt;. It works like onLayout and sends you information about how your content was rendered, after it's rendered. It provides rich information about the text, which you can see in the RNTester example:

![RNTester example](https://raw.githubusercontent.com/mmmulani/image-hosting/master/rntester-textlegend.png)

However, if you want to implement a capheight-baseline bounding box using marginTop and marginBottom, you have to do two render passes. Not only is this slow but it's visually jarring, as the text will appear to move around.

I want to make this render in one pass. Rendering &lt;Text&gt; with a bounding box of capheight-baseline requires knowledge of the size of the ascender, descender and capHeight, which are all derived from the font used. Thus, it is possible to take these values and synthesize a marginTop/marginBottom for the &lt;Text&gt; before Yoga evens looks at the view. I have a couple ideas for the API that I want some feedback on.

My current idea is to do a linear equation of the ascender, descender, capHeight, and xHeight, and allow the user to pass in the corresponding values. For example, if you wanted to render a capheight-baseline bounding box, you would do:

```
<Text
  advancedTextMargin={{
    top: {
      ascender: -1,
      capHeight: 1,
    },
    bottom: {
      descender: -1,
    },
  }}
  >
  This has a capheight-baseline bounding box.
</Text>
```

The values get passed into a simple linear equation like: `margin adjustment = font.ascender * advancedTextMargin.ascender + font.capHeight * advancedTextMargin.capHeight + ...`. This is a bit weird and feels more like a graphics API than a style API but it lets us do this without having to teach Yoga anything.

It's also pretty flexible, suppose you made an edgy logo all in lowercase, and wanted to have a bounding box of just the x-height. You could accomplish it with this:

```
<Text
  advancedTextMargin={{
    top: {
      ascender: -1,
      xHeight: 1,
    },
    bottom: {
      descender: -1,
    },
  }}
  >
  This has a x-height bounding box.
</Text>
```

My other thought for an API would be less expressive but much more understandable:

```
<Text
  style={{
    marginTop: 'x-height',
    marginBottom: 'baseline',
  }}
  >
  This has a x-height bounding box.
</Text>
```
