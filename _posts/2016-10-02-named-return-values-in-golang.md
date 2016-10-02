---
layout: post
title: Do you really know named return value in golang?
date: 2016-10-02
category: work
meta: Be careful when using named return value in golang
---

# Intro

I'm recently writing some tools with golang and found golang function ships a feature called `named return value`. It was not introduced in other common programming languages like Python/Javascript/Java and not exactly the same as that in some 'functional' programing languages like Clojure/Haskell/Scheme which the evaluation of the last expression is the 'return value' of the function. In golang, we can write code like [this](https://tour.golang.org/basics/7):

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

We can leave the variable `x, y` out in the `return` expression. Deadly simple and convinient for short functions just like other golang programs which have their var names short and meaningful. However, today what I'm talking about some common pitfalls while using this awesome feature.

# Pitfall 1.

Considering this problem, given an element `el` and find the  index `idx` in an array (or slice) `arr`. Without `named return value`, you may write code like:

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

Using `named return value`, you can change it to:

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

That's to bad we got an error like:

> Line 12: i is shadowed during return

How could it be? The reason is `scope`.

Golang provides block scope and a pair of `{}` create a new level of function scope. Our range loop body are wrapped in a `{}` plus we are using `:=` to create variables `idx` and `v`. So in the loop body we are totally creating a new variable name `idx` who is `shadowing` the outer scope binding `idx`.

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

The word `shadow` or `binding` are some academic saying, learn more [here](https://en.wikipedia.org/wiki/Variable_shadowing)

# Pitfall 2.

If you are saying pitfall 1 is not a truth since the compiler will complain the wrong code, let's see the second one.

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

In the output line 1, we found the admin user `root` in `us1`, good.

Wait, the output line 2: WTF how does `guest` be an `Admin` users? There are no `Admin` user in `us2` at all!

The reasone is not mysterious: in the range loop, we can't find an `Admin` user in `us2` obviously. But the named value `u` is modified eash iteration in the range loop body. After the loop it was set to the last element in the `us2` array and notice the `return` after the range loop, we are returning it to the caller. So in the `main` function, `admin2` was set to `guest` which is not what we want.

Change the code:

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

The result would be right now.

# Conclusion

1. Golang has block scope which may have a impact on your `named return value`
2. Don not forget to reset the named return value after a range loop

Hope this can help you golang beginner like me :)
