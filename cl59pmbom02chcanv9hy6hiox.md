# Tailwind CSS with CSS-IN-JS



# Introduction
During my early days of front-end development, I was never good at the presentation layer of things, I always find it difficult working with CSS, and it always looked boring to me no matter how many times I look at it🤫. Most front-end developers struggle with designing a simple UI for the little or personal projects they might be working on, during the learning process you wouldn't hire a designer for side projects, most of which wouldn't even make it to production 😂. You'd need the simplest of UI at least to show your work,  Tailwind CSS helps in rapidly building those simple UIs and of course in building full fledged websites.
Over the years many CSS frameworks have been created, CSS Bootstrap probably the most popular among them has helped developers and designers speed up their development process. Tailwind CSS has gained lots of traction, and more and more designers and developers are making use of it in their projects.

If you have worked with Tailwind CSS, you will quickly notice how your classes start to get so large when creating your website, due to its atomic design approach, the way one could reduce this is by creating components in Tailwind CSS or with JavaScript components,  where you might require some interactivity.

There are many UI libraries based on different Javascript frameworks and libraries, most of them use CSS objects to create their styles for each component which is then passed into your chosen CSS-in-JavaScript library, but what if you could have a library that could convert Tailwind CSS classes into CSS objects and still get to use the interactivity of these components. I decided to go hunting if there's an implementation of such and I found an amazing library called [**Twin**](https://github.com/ben-rogerson/twin.macro)


One of the reasons have always been sceptical about CSS-in-Js is the performance bottlenecks when running data-intensive applications, with Twin there's zero runtime code. Popular libraries like radix-ui and headlessui have unstyled UI components, how you style your components is entirely up to you, however, these libraries have a limited set of components. You might still want a good set of UI components that will cater for all your UI needs, you just want to be able to use CSS in these popular UI libraries.

Most UI libraries like [**MUI**](https://mui.com/), [**Chakra UI**](https://mui.com/) and [**Mantine**](https://mantine.dev/) (which I love most) use 'sx' props to style their components, with Twin you can use Tailwind classes to create CSS objects. At compile time Babel grabs these classes and converts them to CSS objects without the need of an extra client-side bundle, leaving no runtime code.
Twin works with many modern  stacks. Most UI libraries use CSS-in-Js libraries like emotion, styled-components, stitches and Goober, and Twin works well with all these libraries. The guys behind it even has some templates where you can kick and start running.

Example.
```javascript
import tw from 'twin.macro'

const Input = ({ hasHover }) => (
  <input css={[tw`border`, hasHover && tw`hover:border-black`]} />
)

```
Now you have both flexibility of CSS-in-Js and the beauty of Tailwind CSS, you can now almost use most of the UI frameworks out there with ease and not worry too much about how to style components of these UI frameworks to fit your application designs.

# Conclusion
CSS-in-JS has dramatically changed the way we author CSS, providing many benefits and improving the overall development experience.
However, choosing which library to adopt is not straightforward and all choices come with many technical compromises, mostly based on how you intend to style its components, Twin will to some extent bring down these compromises and you'll be able to continue to use your preferred library.
