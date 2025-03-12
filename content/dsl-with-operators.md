Title: Building a DSL with Python Operators
Date: 2025-03-12
Category: Blog

I've been a little obsessed with operator overloading lately. First using `|=` in [sqlalchemy-builder]({filename}/sqlalchemy-footgun.md) and then using `|` and `@` in [better-functools]({filename}/fun-with-functional.md). 

I didn't actually know that these qualify as DSLs, specifically what's known as "internal DSLs". Funnily enough, I'm usually not a fan of DSLs. A few reasons come to mind:

- DSLs are most often half baked, pretty much every CI-CD pipeline uses a YAML based DSL that's frustrating to work with.
- DSLs lack the IDE support of a proper language.
- DSLs are another thing to learn, internal DSLs are especially annoying as you need to learn the host language too.

But I realise now that DSLs are also an opportunity to introduce something new in a familiar environment. To demonstrate how and why, I've created another cookbook that builds a functional DSL from the ground up.


<iframe
  src="https://jamie-chang.github.io/cookbooks/notebooks/index.html?path=operator.ipynb"
  width="100%"
  height="800em"
  style="border: 1px solid #EEEEEE;"
></iframe>

