# Why you should consider TypeScript for your next React project

For the past couple of years have always been a user of programming languages that are strongly typed, used C#(still my best), Java, Kotlin etc. you have to declare the exact data types for your variable, as such you run into less errors at compile time. When I started writing codes in Javascript things seemed simple enough until you’d run into some funny bugs and you find yourself trying to solve a crime in which you are the culprit, lol. There comes in Typescript, and guess what the same guy behind C# created Typescript, thanks for all you do sir, Anders Hejlsberg  “salute”.
I guess he was frustrated like I was with some parts of javascript, lol. Ask around most errors found with javascript are mostly null errors and undefined errors, with static type checking Typescript could help avoid this heartbreaking moments.
Let’s take a loot at an example using Next(React) with Typescript.
Consider this PetCard component, that takes props age,location and image.
Let’s assume this is the structure of expected data:
```js
{
	Age: ‘2’,
	location:’Mars’,
	images:[‘image1.jpg’,’image2.png’],
}
``` 
**The componet without TypeScript**
```js
const PetCard = props  => {
   const {age,location,image} = props //Destructing
    return <>
      ///... rest of code
    </>;
};
``` 
**Passing the props**
```js
{data && data.pets?.map(pet, index) => (
              <div key={index} className="key">
                <PetCard {...pet} />{/* Spread syntax */}
              </div>
            ))}
```  
With name of the "image" I forgot and assumed I was expecting a single string whereas it was an array of string, can’t blame me, am only human, lol . To JavaScript, at compile time it's fine but at runtime you'd get an error. console.log to the rescue, it was an array of string not a string after logging it to the console. The error message from 'next/image' component didn't help either :(.
![Screenshot 2022-02-02 231103.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643814806719/nZIlivzru.png)

```js
<Image
     className="w-auto h-48"
     src={img}
      height="192"
      width="192"
      alt="Pet image"
 </>
``` 
I was pretty sure I had the src property for the Image component.

Lets use TypeScript.
With TypeScript you must declare your data types, that's the rule.

```js
export type PetCardPropsType = {
    age:number,
    location:string,
    image:string[],
}
//You can use an Interface too for type declaration
``` 

```js
{data && data.pets?.map(({ ...pet }: PetCardPropsType, index: number) => (
              <div key={index} className="key">
                <PetCard {...pet} />
              </div>
            ))}
``` 
Even with the misspelled name of the prop images as image, not much of big deal, but it helps when you give proper names to your variable, as long as it's type is declared as an array of string(image:string[]). Using the type on data.pets mapping ensures what you are expecting conforms with structure of the PetCardPropsType, every mapped data must have the exact data type as described by the type.
Component with Typescript:
```js
export type PetCardPropsType = {
    age:number,
    location:string,
    image:string[],
}
const PetCard: React.VFC<PetCardPropsType> = ({name,animal,city,images,state}) => {
   
    }
    return <>
 //..rest of the code.
}
``` 
**Conclusion**
It's absolutely fine using just JavaScript in your projects, I still do mostly for side and small projects, most developers still do, most libaries and frameworks still use JavaScript but more libaries and frameworks are jumping in on the wagon, TypeScript requires more typing but can reduce less debugging, and we sometimes spend more time debugging than developing. As projects get bigger you need every possbile tools available to reduce bugs in projects.

The ecosytem keeps getting bigger, In recent months, the amount of packages that use TypeScript seemed higher than before. Browsing and reading TypeScript with knowing just JavaScript is definitely possible, but quickly scanning those is pretty easier now.


