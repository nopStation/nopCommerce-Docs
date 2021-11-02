---
title: মাইগ্রেশন কিভাবে কাজ করে?
uid: en/developer/tutorials/migrations
author: git.AndreiMaz
contributors: git.AfiaKhanom
---
# মাইগ্রেশন কিভাবে কাজ করে?

## ডাটাবেসের সাথে কাজ করার পদ্ধতির পরিবর্তনের সংক্ষিপ্ত বিবরণ

ডাটাবেসের সাথে কাজটি নপকমার্স সংস্করণ ৪.৩০ তে উল্লেখযোগ্যভাবে পুনর্নির্মাণ করা হয়েছিল। প্রথম পরিবর্তন যা লক্ষ্য করা যেতে পারে তা হ'ল নেভিগেশন প্রোপের্টগুলির সম্পূর্ণ প্রত্যাখ্যান। আমরা এই পদ্ধতির উপযোগিতা নিয়ে ভাবতে পারি এবং তর্ক করতে পারি কিন্তু এর অবশ্যই কয়েকটি ইতিবাচক বিষয় রয়েছে:

১. কোড বোঝা এবং রক্ষণাবেক্ষণ সহজ করুন।
 > [!NOTE]
 > কোড রিফ্যাক্টরিং চলাকালীন, আমরা কর্মক্ষমতা এবং কার্যকারিতা উভয়কে প্রভাবিত করে বেশ কয়েকটি ত্রুটি খুঁজে পেয়েছি এবং সংশোধন করেছি।
২. প্রশ্নের উপর সম্পূর্ণ নিয়ন্ত্রণ এবং তাদের বাস্তবায়নের মুহূর্ত (যা ইতিবাচকভাবে সমগ্র সমাধানের কর্মক্ষমতা প্রভাবিত করে)।

৩. যেকোনো ডাটাবেস কাঠামোতে মাইগ্রেশন প্রক্রিয়া সহজ করার সম্ভাবনা (সবচেয়ে গুরুত্বপূর্ণভাবে)।

যেহেতু নপকমার্স সম্পূর্ণরূপে .Net Core (সংস্করণ ৪.১০) এ স্যুইচ করেছে এবং একটি ক্রস-প্ল্যাটফর্ম সমাধান হয়ে উঠেছে, তাই বেশ কয়েকটি ডাটাবেসকে সমর্থন করা আরও গুরুত্বপূর্ণ বিষয় হয়ে ওঠে। নপকমার্স টিম যথেষ্ট গবেষণা এবং বিশ্লেষণ করেছে এবং স্ট্যান্ডার্ড সত্তা ফ্রেমওয়ার্ক কোর ব্যবহার বন্ধ করার সিদ্ধান্ত নিয়েছে। একই সময়ে, আমরা OOP পদ্ধতি ব্যবহার করে LINQ প্রশ্নের মাধ্যমে ডাটাবেসের সাথে কাজ না করার সিদ্ধান্ত নিয়েছি (C#-ডেভেলপারদের দ্বারা ব্যবহৃত সবচেয়ে সাধারণ পদ্ধতি কী)। চূড়ান্ত পছন্দ Linq2DB এবং FluentMigrator এর একটি গুচ্ছের উপর পড়ে। নীচে, আমি এই প্রতিটি কাঠামোর ভূমিকা বিশদভাবে বর্ণনা করব।

## Linq2DB

> [!NOTE]
> ৪.৩০ সংস্করণ থেকে শুরু নপকমার্স একটি ORM ফ্রেমওয়ার্ক হিসাবে Linq2DB ব্যবহার করে। Linq2DB হল একটি অবজেক্ট-রিলেশনাল ম্যাপার (ORM) যা .NET ডেভেলপারদের .NET অবজেক্ট ব্যবহার করে একটি ডাটাবেসের সাথে কাজ করতে সক্ষম করে। এটি বিভিন্ন সংখ্যক ডাটাবেস প্রদানকারীর কাছে নেট বস্তুর মানচিত্র তৈরি করতে পারে।

নপকমার্সে, Linq2DB একটি ডাটাবেস-অ্যাক্সেস স্তর হিসাবে ব্যবহৃত হয়। বর্তমানে, নপকমার্স দুটি জনপ্রিয় ডাটাবেস সমর্থন করে: MS SQL সার্ভার এবং MySQL সার্ভার। যদি আমরা কোডটি বিশ্লেষণ করি, আমরা সহজেই দেখতে পাচ্ছি যে প্রতিটি ডাটাবেস তার নিজস্ব শ্রেণী দ্বারা সমর্থিত যা INopDataProvider ইন্টারফেস প্রয়োগ করে। কিন্তু আপনি যদি নিজের ডাটাবেস অ্যাক্সেস প্রদানকারী তৈরির পরিকল্পনা না করেন, তাহলে আপনি বাস্তবায়নের বিবরণ একেবারেই উপেক্ষা করতে পারেন। বেশিরভাগ উন্নয়নমূলক কাজের জন্য, কয়েকটি পয়েন্ট বোঝা যথেষ্ট হবে:

 ১. ডাটাবেজে (POCO ক্লাস) টেবিলের সাথে সম্পর্কিত একটি বস্তুর প্রয়োজন।

 ২. টেবিল ডেটা সহ সমস্ত কাজ ইরিপোজিটরি `<TEntity>` ইন্টারফেসের মাধ্যমে সম্পন্ন করা হয়। এমনকি আইওসিতে এটির স্থাপনার যত্ন নেওয়ারও দরকার নেই, কারণ এটি উপযুক্ত কারখানা মেথডতে একটি কলের মাধ্যমে নিবন্ধিত।

 ৩. ডাটাবেসে টেবিলের সৃষ্টি নিয়ন্ত্রণ করতে হবে।

এবং শেষ সমস্যা সমাধানের জন্য, আমাদের বান্ডিল থেকে দ্বিতীয় কাঠামো মোকাবেলা করতে হবে, যথা FluentMigrator।

## FluentMigrator

> [!NOTE]
> Fluent Migrator হল .NET- এর জন্য একটি মাইগ্রেশন ফ্রেমওয়ার্ক যা অনেকটা রুবি এর রেল মাইগ্রেশনের মত। *Migrations* আপনার ডাটাবেস স্কিমা পরিবর্তন করার একটি কাঠামোগত উপায় এবং এটি প্রচুর এসকিউএল স্ক্রিপ্ট তৈরির বিকল্প যা প্রতিটি ডেভেলপারকে ম্যানুয়ালি চালাতে হবে। মাইগ্রেশন একাধিক ডাটাবেসের জন্য একটি ডাটাবেস স্কিমা বিকশিত করার সমস্যার সমাধান করে (উদাহরণস্বরূপ, ডেভেলপারের স্থানীয় ডাটাবেস, পরীক্ষা ডাটাবেস এবং উৎপাদন ডাটাবেস)। ডাটাবেস স্কিমা পরিবর্তনগুলি C# এ লেখা ক্লাসগুলিতে বর্ণিত হয়। এই ক্লাসগুলি একটি সংস্করণ নিয়ন্ত্রণ ব্যবস্থায় পরীক্ষা করা যেতে পারে।

আপনার সত্তা যুক্ত করার বিস্তারিত পরিকল্পনা নিম্নলিখিত নিবন্ধে বর্ণনা করা হয়েছে: [ডেটা অ্যাক্সেস সহ প্লাগইন](xref:bn/developer/plugins/how-to-write-plugin-4.30)। অতএব, আমরা কেবল সাধারণ তাত্ত্বিক বিষয়গুলিতে থাকব:

১. মাইগ্রেশনগুলি নপকমার্স কোডের স্তরেই সমর্থিত।

২. আপনি অ্যাবস্ট্রাক্ট ক্লাস **MigrationBase** থেকে উত্তরাধিকারসূত্রে প্রাপ্ত যেকোন মাইগ্রেশন তৈরি করতে পারেন।

৩. মাইগ্রেশনের জন্য সংস্করণ নিয়ন্ত্রণ সহজ করার জন্য আমরা কোডে **MigrationAttribute** থেকে উত্তরাধিকারসূত্রে প্রাপ্ত **NopMigrationAttribute** বৈশিষ্ট্য যোগ করেছি। এখন আপনি কেবল সাধারণ তারিখের পরিবর্তে তারিখ এবং সময় নির্দিষ্ট করতে পারেন যখন মাইগ্রেশন তৈরি করা হয়েছিল।

৪. আমরা **SkipMigrationOnUpdateAttribute** অ্যাট্রিবিউটও যোগ করেছি যা নির্দেশ করে যে আপডেট প্রক্রিয়ার সময় মাইগ্রেশন এড়িয়ে যাওয়া উচিত কিনা।

৫. আপনি ডাটাবেসে দুটি উপায়ে একটি টেবিল তৈরি করতে পারেন:
 * আপনার মাইগ্রেশন ক্লাসের **Up** মেথডতে **Create.Table** মেথড ব্যবহার করুন এবং এক্সটেনশন মেথড ব্যবহার করে সমস্ত বিবরণ নির্দিষ্ট করুন।
 * আপনার মাইগ্রেশন ক্লাসের **Up** মেথডে **IMigrationManager.BuildTable\<T\>** মেথড ব্যবহার করুন এবং **IEntityBuilder** এবং **INameCompatibility** এর প্রয়োগ ব্যবহার করে প্রয়োজন হলে সমস্ত বিবরণ উল্লেখ করুন ইন্টারফেস (নপকমার্সে আমরা এই পদ্ধতি ব্যবহার করি)।