---
layout: post
title: Do you really know named return value in golang?
date: 2016-10-02
category: work
meta: Be careful when using named return value in golang
---

# Intro

Recently, I'm writing some golang tools and have found that functions of golang have a feature called `named return value`, which cannot be found in other common programming languages like Python, Javascript or Java. It cannot also be equal to the feature in some 'functional' programing languages like Clojure, Haskell or Scheme, which the evaluation of the last expression is the 'return value' of the function. In golang, we write code like [this](https://tour.golang.org/basics/7):

{% highlight go %}
package main

import "fmt"

func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}

func main() {
	fmt.Println(split(17))
}
{% endhighlight %}

By leaving out the variable `x, y`in the `return` expression, we have created a shorter function which already permit precise variable name in golang. Fairly simple and convenient. However, what I'm talking about today are some common pitfalls when using this handy feature.

# Pitfall 1.

Considering this problem: find the index `idx` of the element "el" in an array (or slice) `arr`. Without `named return value`, you may write code like this :

{% highlight go %}
func findIdx(arr []int, el int) int {
  for idx, v := range arr {
    if v == el {
      // found you!
      return idx
    }
  }
  return -1
}
{% endhighlight %}

Using `named return value`, you can change it to this:

{% highlight go %}
func findIdx(arr []int, el int) (idx int) {
  for idx, v := range arr {
    if v == el {
      // found you!
      return
    }
  }
  return -1
}
{% endhighlight %}

[Let's run it to see what happens](https://go-sandbox.com/#/1U6U9Aadnh).

We encountered the following error:

> Line 12: i is shadowed during return

How could it be? The reason is `scope`.

In Golang, one type of block scope is a pair of `{}` which generates a new level of function scope. In our example, our range loop body is wrapped in a `{}`, and, at the same time, we are also using `:=` to assign variables `idx` and `v`. Therefore, we have completely declared another variable name "idx", which is "shadowing" the outer scope binding (our intended)'idx' in the loop body. You can learn about the academic meaning of the word `shadow` or `binding` [here](https://en.wikipedia.org/wiki/Variable_shadowing)

We can change the code to:

{% highlight go %}
func findIdx(arr []int, el int) (idx int) {
  for i, v := range arr {
    if v == el {
      // found you!
      // note: no colon here
      idx = i
      return
    }
  }
  return -1
}
{% endhighlight %}



# Pitfall 2.

If you are confident that you won't fall into the first case, since the compiler will warn of the error. Pitfall 2 is less obvious then.


{% highlight go %}
type User struct {
	Admin bool
	Name  string
}

func findAdmin(users []*User) (u *User) {
  // note: not colon, aka not shadowling :)
	for _, u = range users {
		if u.Admin {
			return
		}
	}
	return
}

func main() {
	var us1 = []*User{
		&User{Admin: false, Name: "anonymouse"},
		&User{Admin: true, Name: "root"},
	}

	admin := findAdmin(us1)
	fmt.Println("Admin: ", admin.Name)

	var us2 = []*User{
		&User{Admin: false, Name: "anonymouse"},
		&User{Admin: false, Name: "guest"},
	}

	admin2 := findAdmin(us2)
	fmt.Println("Admin: ", admin2.Name)
}
{% endhighlight %}

[Run it](https://go-sandbox.com/#/8m_gOo-FYi)

Output:

> Admin:  root
> Admin:  guest

> Program exited.

In line 1 from output, we confirmed that the admin user `root` is in `us1`, good.

But wait. Look at line 2: There is no `Admin` user in `us2` at all! How could `guest` turn out to be an `Admin` users?

Well, now, we shouldn't find it hard to figure it out:  Although the named value `u` is modified during each iteration in the range loop body, it will, at the end of loop, be set to the last element in the `us2` array. Note the `return` after the range loop. It will then return the value to the caller. As a result, in the `main` function, `admin2` is set to `guest`. This is clearly not what we intended.

To correct the code:

{% highlight go %}
func findAdmin(users []*User) (u *User) {
  // note: not colon, aka not shadowling :)
	for _, u = range users {
		if u.Admin {
			return
		}
	}
  u = nil
	return
}
{% endhighlight %}

Now, it will end up with the right outcome.

# Conclusion

1. Beware of variable shadow and binding within a golang range loop
2. Double check the named return value after writing a range loop

Hope my little reflection may help some of you Gophers. :)
