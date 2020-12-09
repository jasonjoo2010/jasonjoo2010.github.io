Visual Studio Code Becomes Laggy/Freezing in Big Sur
===

- date: 2020-12-09
- tags: Big Sur

------------

# Issue

After upgrading Big Sur some applications became freezing or laggy. Especially in Visual Studio Code the terminal became very laggy typing commands. Even unusable.

# Solution

This solution works to me.

```shell
codesign --remove-signature /Applications/Visual\ Studio\ Code.app/Contents/Frameworks/Code\ Helper\ \(Renderer\).app
```

Then reopen it.

Maybe it also works to other similar issued applications.


