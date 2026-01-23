---
layout: post
title: "Old style imperative mocks vs moq functional specifications"
date: 2010-09-22 15:56:00 +0000
tags: [ddd, ef, eventsourcing, extensibility, gadgets, mocking, moq, msbuild, mvc, nuget, patterns, programming, t4, technology, vsx, wcf, webapi]
---

##  [Old style imperative mocks vs moq functional specifications](http://blogs.clariusconsulting.net/kzu/old-style-imperative-mocks-vs-moq-functional-specifications/ "Old style imperative mocks vs moq functional specifications")

September 22, 2010 3:56 pm

Here’s a fairly complex object graph that needs to be setup as part of a unit test, as done in “traditional” [moq](http://moq.me/):
    
    
        var el1 = new Mock<IElementInfo>();
        el1.Setup(x => x.Id).Returns(Guid.NewGuid());
        el1.Setup(x => x.Multiplicity).Returns(Multiplicity.Single);
    
        var c1 = new Mock<ICollectionInfo>();
        c1.Setup(x => x.Id).Returns(Guid.NewGuid());
        c1.Setup(x => x.Multiplicity).Returns(Multiplicity.Multiple);
    
        var p1 = new Mock<IPropertyInfo>();
        p1.Setup(x => x.Id).Returns(Guid.NewGuid());
        p1.Setup(x => x.Name).Returns("Foo" + Guid.NewGuid().ToString());
        p1.Setup(x => x.Type).Returns("System.String");
    
        var p2 = new Mock<IPropertyInfo>();
        p2.Setup(x => x.Id).Returns(Guid.NewGuid());
        p2.Setup(x => x.Name).Returns("Bar" + Guid.NewGuid().ToString());
        p2.Setup(x => x.Type).Returns("System.String");
    
        var elementInfoMock = new Mock<IElementInfo>();
        elementInfoMock.Setup(e => e.Id).Returns(Guid.NewGuid());
        elementInfoMock.Setup(e => e.Multiplicity).Returns(Multiplicity.Multiple);
        elementInfoMock.Setup(e => e.Elements)
            .Returns(new List<IAbstractElementInfo>
            {
                el1.Object,
                c1.Object,
            });
        elementInfoMock.Setup(x => x.Properties).Returns(
            new List<IPropertyInfo>
            {
                p1.Object,
                p2.Object,
            });
    
        this.elementInfo = elementInfoMock.Object;
    

Things to note (familiar to any moqer): 

  1. Multiple temporary variables for the sole purpose of setting up intermediate mocks 
  2. Setup…Returns repetitive (and **boring**!) pattern 
  3. mock.Object annoying indirection to the mocked instance, ’cause we have to do so much to it **before** we’re done with it 

Well, I know I got tired of all that crap. So in moq v4, you can reduce all that to a much more readable and fluent moq functional specification like this:
    
    
    this.elementInfo = Mock.Of<IElementInfo>(x =>
        x.Id == Guid.NewGuid() &&
        x.Multiplicity == Multiplicity.Multiple &&
        x.Elements == new List<IAbstractElementInfo>
        {
            Mock.Of<IElementInfo>(e => e.Id == Guid.NewGuid() && e.Multiplicity == Multiplicity.Single),
            Mock.Of<ICollectionInfo>(e => e.Id == Guid.NewGuid() && e.Multiplicity == Multiplicity.Single),
        } &&
        x.Properties == new List<IPropertyInfo>
        {
            Mock.Of<IPropertyInfo>(p => p.Id == Guid.NewGuid() && p.Name == "Foo" + Guid.NewGuid() && p.Type == "System.String"),
            Mock.Of<IPropertyInfo>(p => p.Id == Guid.NewGuid() && p.Name == "Foo" + Guid.NewGuid() && p.Type == "System.String"),
        });

What you're essentially saying is "Give me a mock that behaves like this" (you also have Mocks.Of<T> if you need many). How's that for improved readability and maintainability?

Haven’t you become a moqer yet? ![;\)](https://web.archive.org/web/20200808212749im_/http://blogs.clariusconsulting.net/kzu/wp-includes/images/smilies/icon_wink.gif)

![](https://web.archive.org/web/20200808212749im_/http://www.clariusconsulting.net/aggbug.aspx?PostID=284965)

/kzu
