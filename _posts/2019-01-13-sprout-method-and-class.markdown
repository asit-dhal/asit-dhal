---
layout: post
title:  "Sprout Method and Class"
date:   2019-01-13 20:00:47 +0200
categories: code-refactoring
comments: true
---

Let's say you have a class called TransactionGate that posts entries in another class called TransactionBundle.


{% highlight cpp %}

class TransactionGate
{
public:
  void postEntries(std::list<Entry> entries)
  {
    for(auto itr = std::begin(entries); itr != std::end(entries); itr++) {
      itr->updatePostDate();
    }

    transactionBundle->getListManager()->add(entries);
  }
};

{% endhighlight %}


There is a new requirement to introduce a check that none of the new entries are already in the bundle.

{% highlight cpp %}

class TransactionGate
{
public:
  void postEntries(std::list<Entry> entries)
  {
    std::list<Entry> entriesToAdd;
    for(auto itr = std::begin(entries); itr != std::end(entries); itr++) {
      if (transactionBundle->getListManager()->hasEntry(*itr)) {
        itr->updatePostDate();
        entriesToAdd.push_back(*itr);
      }
    }

    transactionBundle->getListManager()->add(entries);
  }
};

{% endhighlight %}

The new changes are pretty invasive. Because we are migling two operations here: date posting and duplicate entry detection. And also the flow is not really sequential.

So, we can add a new method to find unique entries. Call that method in the old method. This will add less code to the existing method.

{% highlight cpp %}

class TransactionGate
{
public:
  std::list<Entry> uniqueEntries(std::list<Entry> entries)
  {
    std::list<Entry> resultEntries;
    for(auto itr = std::begin(entries); itr != std::end(entries); itr++) {
      if (transactionBundle->getListManager()->hasEntry(*itr)) {
        resultEntries.push_back(*itr);
      }
    } 
  }

  void postEntries(std::list<Entry> entries)
  {
    std::list<Entry> entriesToAdd = uniqueEntries(entries);

    for(auto itr = std::begin(entriesToAdd); itr != std::end(entriesToAdd); itr++) {
      itr->updatePostDate();
    }

    transactionBundle->getListManager()->add(entriesToAdd);
  }
};

{% endhighlight %}

This method of refactoring code is called *sprout method*.


Here is an old method in a class called QuarterlyReportGenerator.

{% highlight cpp %}

class QuarterlyReportGenerator
{
public:
  std::string generate()
  {
    std::vector<Result> results = database.queryResults(beginDate, endDate);

    std::string pageText;
    pageText += "<html><head><title>"
          + "Quaretly Report"
          + "</title></head>"
          + "<body><table>";

    if (!results.empty()) {
      for(auto itr = std::begin(results); itr != std::end(results); itr++) {
        pageText += "<tr>";
        pageText += "<td>" + itr->department + "</td>";
        pageText += "<td>" + itr->manager + "</td>";
        char buffer[128];
        sprintf(buffer, "<td>$%d</td>", itr->netProfit/100);
        pageText += std::string(buffer);
        sprintf(buffer, "<td>$%d</td>", itr->operatingExpense/100);
        pageText += std::string(buffer);
        pageText += "</tr>";
      }
    } else {
      pageText += "No results for this period";
    }
    pageText += "</table></body></html>";

    return pageText;
  }
};

{% endhighlight %}

Let's say there is a requirement to add a header row. Adding more code to this method will make it more difficult manage.

We can create a new class for this and reuse this class in the existing implementation.

{% highlight cpp %}

class QuarterlyReportTableHeaderProducer
{
public:
  std::string makeHeader()
  {
    return "<tr><td>Department</td><td>Manager</td><td>Profit</td><td>Expense</td></tr>";
  }
};

{% endhighlight %}

In QuarterlyReportGenerator class

{% highlight cpp %}

QuarterlyReportTableHeaderProducer producer;
pageText += producer.makeHeader();

{% endhighlight %}

We can document that commonality in the code by creating an interface class and having them both inherit from it.

{% highlight cpp %}

class HTMLGenerator
{
public:
  virtual ~HTMLGenerator() = nullptr;
  virtual std::string generate() = nullptr;
};

class QuarterlyReportTableHeaderProducer : public HTMLGenerator
{
public:
  std::string makeHeader() override
  {
    // implementation
  }
};

class QuarterlyReportGenerator : public HTMLGenerator
{
public:
  std::string generate() override;
  {
    // implementation
  }
};

{% endhighlight %}

By using these steps, we can fold the class into the set of concepts that we already had in the application. 

This method of refactoring is called sprout class. 

> Sprout method are recommended whenever you can see the code that you are adding as a distinct piece of work. Sprout class are recommended whenever you can see that the code you are adding as a distinct repsonsibility.

