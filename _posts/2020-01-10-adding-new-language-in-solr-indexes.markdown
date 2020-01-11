---
layout: post
title:  "Adding new language in SOLR indexes"
date:   2020-01-10 13:51:51 +0300
categories: blog sitecore
---
Sometimes you may have language in Sitecore that doesn't exist in SOLR which may lead to the errors like `unknown field 'title_t_pl'` during rebuild of index:
{% highlight xml %}
Exception: SolrNet.Exceptions.SolrConnectionException
Message: <?xml version="1.0" encoding="UTF-8"?>
<response>
<lst name="responseHeader"><int name="status">400</int><int name="QTime">88</int></lst><lst name="error"><lst name="metadata"><str name="error-class">org.apache.solr.common.SolrException</str><str name="root-error-class">org.apache.solr.common.SolrException</str></lst><str name="msg">ERROR: [doc=sitecore://web/{2bc1bf75-9072-463f-a7d3-3c69a006a539}?lang=pl-pl&amp;ver=1&amp;ndx=sitecore_web_index] unknown field 'title_t_pl'</str><int name="code">400</int></lst>
</response>
{% endhighlight %}
Postfix `_pl` means Polish language and this error says that there is no such language configured in SOLR.
In this article will be provided solution of this problem. Basically it is combined from posts of <a href="https://sitecore.namics.com/2018/11/26/adding-new-languages-to-sitecores-solr-indexes/">Fabian Geiger</a>, <a href="https://mikael.com/2018/01/working-with-content-search-and-solr-in-sitecore-9/">Mikael Högberg</a> and <a href="https://sitecoreblog.marklowe.ch/2018/10/customize-solr-managed-schema/">Mark Lowe</a> posts. Thank you guys!

Our target is to add two new elements in SOLR `managed-schema` file:
<script src="https://gist.github.com/alexvolchetsky/da9cebc353d6c19a293e12382e7c83e7.js"></script>
This file could be found here: `%YourSolrInstancePath%\server\solr\%YourIndexName%\conf\managed-schema`

You can add `fieldType` and `dynamicField` elements mentioned above in this file manually but this is not recommended as these changes will be discarded on next population of schema. Here we have better solution.

We will extend `contentSearch.PopulateSolrSchema` pipeline by adding new custom processor. Also we will put new `customSolrManagedSchema` section in config file with nested `commands` element. This element will contain commands that will be send to SOLR during population of managed schema. Here we have two commands: `add-field-type` and `add-dynamic-field`. First one will create `text_pl` field type for Polish language. And the second one will create dynamic field `*_t_pl`. It will help SOLR to recognize fields like `title_t_pl` and correctly index them. You can see below example of mentioned config file. Also notice `applyToIndexes` attribute has index names for which commands will be executed.
<script src="https://gist.github.com/alexvolchetsky/6977f40f662be0eae45e93351fec7506.js"></script>

`CustomPopulateManagedSchema` processor is inherited from `Sitecore.ContentSearch.SolrProvider.Pipelines.PopulateSolrSchema.PopulateFields` class. Its `GetHelper()` method is overriden to return our helper `CustomSchemaPopulatorHelper`.
<script src="https://gist.github.com/alexvolchetsky/d8cdd0f33983d0ea8f4a63da5a13b1d1.js"></script>

`CustomSchemaPopulatorHelper` will read commands from `customSolrManagedSchema/commands` config section. It will create additional command to delete field type in SOLR schema before execution of `add-field-type` if this field type already exists in SOLR schema. For `add-dynamic-field` command there is no additional processing.
<script src="https://gist.github.com/alexvolchetsky/3615f0e767dd976bbb35659723e684cd.js"></script>

After helper will return list of commands in XML they will be serialized to JSON under the hood and then sent to SOLR. Here are examples of JSON formats for commands <a href="https://lucene.apache.org/solr/guide/6_6/schema-api.html">https://lucene.apache.org/solr/guide/6_6/schema-api.html</a>

After all changes are implemented and published we can try to populate SOLR managed schema. Open Control Panel -> Populate Solr Managed Schema -> keep selected indexes for which commands from `customSolrManagedSchema` should be applied and click Populate. After that check if `managed-schema` has `text_pl` fieldType and `*_t_pl` dynamicField.
But at this point it is possible that this fields will not be added. This is because SOLR can't recognize some of our `filters` we added in `customSolrManagedSchema`. To check if this is true you can comment out all `filters` elements in config and then repeat schema population. In that case our new fieldType and dynamicFields should appear in `managed-schema`. It is only left make it to recognize our filters. 

For every index need to:
1. Go to `%YourSolrInstancePath%\server\solr\%YourIndexName%\conf\lang` and create `stopwords_pl.txt` file. Restart SOLR service. Uncomment first two `filters` elements but keep last element commented. Repeat schema population process. Now you will see two `filters` in `managed-schema` file.
2. Open `%YourSolrInstancePath%\server\solr\%YourIndexName%\conf\solrconfig.xml` and add `<lib dir="${solr.install.dir:../../../..}/contrib/analysis-extras/lucene-libs" regex=".*\.jar" />`. This change will enable `org.apache.lucene.analysis.stempel.StempelPolishStemFilterFactory`. Restart SOLR service. Uncomment last `filters` element. Repeat schema population. Now you will see three `filters` in `managed-schema` file.

Of course this is not mandatory to comment out `filters` elements but it helps to troubleshoot issues.

After that SOLR will have Polish language added. For other languages process will be similar. In <a href="https://sitecore.namics.com/2018/11/26/adding-new-languages-to-sitecores-solr-indexes/">Fabian Geiger</a> post you can find values in `managed-schema` for other languages.
Now you can try rebuild index and error will disappear.

Happy coding!