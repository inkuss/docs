
---
title: "Courses"
linkTitle: "Courses"
date: 2021-02-23
weight: 40
---

The Courses app allows you to create and manage course reserves.

Note: In order for the courses you create in the Courses app to be discoverable by your patrons, you need to have an external interface or discovery layer set up and capable of interacting with FOLIO.


## Permissions

The permissions listed below allow you to interact with the Courses app and determine what you can or cannot do within the app. You can assign permissions to users in the Users app. If none of these permissions are assigned to a user, they are unable to see the Courses app or any related information.

The following are all the Courses permissions, listed from least to most restrictive:

* **Courses: All permissions.** This permission allows the user to maintain (view, add, edit, and delete) courses, items, instructors, and cross-listed courses.
* **Courses: Read, Add, edit.** This permission allows the user to view, add, and edit a course. However, they are unable to delete a course.
* **Courses: Add, edit, and remove items.** This permission allows the user to view, add, edit, and remove items associated with a course.
* **Courses: Read All.** This permission allows the user to see all courses and item information.
* **Settings (Courses): Can create, edit and delete Course Settings.** This permission allows the user to maintain (view, add, edit, and delete) all course settings.
* **Settings (courses): display list of settings pages.** This permission allows the user to access and view course settings. However, they are unable to add, edit, or delete any settings.
* **UI: courses module is enabled.** This permission adds the Courses app icon to the user's FOLIO toolbar. It does not give the user the ability to view or interact with the Courses app or settings.


## Implementation considerations

Before you implement the Courses app, make sure you have completed the following:

* Implemented the Inventory app.
* [Configured your circulation rules.]({{< ref "/settings_circulation.md" >}})
* Loaded or created users.

If you are configuring the Courses app for the first time, you need to first set up the following features in the Settings app, if applicable:

* [Terms]({{< ref "/settings_courses.md#settings--courses--terms" >}})
* [Course Types]({{< ref "/settings_courses.md#settings--courses--course-types" >}})
* [Course Departments]({{< ref "/settings_courses.md#settings--courses--course-department" >}})
* [Processing Statuses]({{< ref "/settings_courses.md#settings--courses--processing-statuses" >}})
* [Copyright Statuses]({{< ref "/settings_courses.md#settings--courses--copyright-statuses" >}})

Once you configure the above settings, you can:

* [Create courses.](#creating-a-course)
* [Add instructors.](#adding-an-instructor-to-a-course) 
* [Add cross-listed courses.](#adding-a-cross-listed-course)
* [Add reserves to courses.](#adding-a-reserve-item-to-a-course)


## Integrations

The Courses app can be integrated with these applications:

* EBSCO Discovery Service (EDS)
* VuFind

In addition, you can connect the Courses app to your learning management system using the Learning Tools Interoperability (LTI) protocol. There is a separate module to install for LTI support. For more information, see [Course Reserves - LTI connectivity](https://wiki.folio.org/display/FOLIOtips/Course+Reserves+-+LTI+connectivity).

If you are implementing the Courses app with any of these applications, each have their own features to consider in regards to the migration of courses, sections, cross-listings, and separate courses and how they interact with FOLIO.


## Searching for courses

You can search for courses in the **Search & filter** pane. All courses are shown and selected by default. To search for courses, enter your search terms into the box. Select the **All fields** drop-down list to search through one of the following fields: Course name, Course code, Section, Instructor, Registrar ID, and External ID. All fields is the default search.

You can also search for courses by selecting any of the filters in the **Courses Search & filter** pane: Department, Course type, Term, and Location. Additionally, you can apply the filters after you perform a search to limit your results.


## Searching for reserves

You can search for items on reserve in the **Search & filter** pane. Click **Reserves **to start your search. Courses are shown and selected by default. To search for reserves, enter your search terms into the box. Select the **All fields** drop-down list to search through one of the following fields: Title, Barcode, or Call Number. All fields is the default search.

You can also search for reserves by selecting any of the filters in the **Search & filter** pane: Processing status, Copyright status, Permanent location, Temporary location, and Term. Additionally, you can apply the filters after you perform a search to limit your results.


## Creating a course

When creating a course, you should keep the following in mind:

* You must have the Courses window open in order to create a course.
* Once a course is created, it can only be deleted if all reserve items are removed. 
* Department, Course Type, and Term are configured in Settings. See [Settings > Courses]({{< ref "/settings_courses.md" >}}) for more information.
* If you are adding one or more cross-listed course to a course, the information you enter into Course listing information also applies to each cross-listed course.
* Reserve items added to the course are automatically assigned with the Start Date and End Date of the Term you selected, as specified in the [Term settings.]({{< ref "/settings_courses.md#settings--courses--terms" >}}) If needed, you can edit the dates by [editing the reserve item.](#editing-a-reserve-item)
* Any item assigned to a Course automatically has its temporary location set to the value specified in the Location field. If needed, you can change the temporary location by [editing the reserve item.](#editing-a-reserve-item)
* When completing the course information, make sure you understand how the fields correspond to your discovery interface.

1. Click **New**.

2. In the **Create course** window, enter a **Course Name** and select a **Term**. All other fields are optional.

3. Click **Save & close**.


## Editing a course

1. [Find the course](#searching-for-courses) you want to edit and click on it in the **Courses **list.

2. In the **course details** window, click **Edit**.

3. Make your desired changes to the course and click **Save & close**.


## Deleting a course

Courses can only be deleted once all items are removed from the course.

1. [Find the course](#searching-for-courses) you want to delete and click on it in the **Courses **list.

2. In the **course details** window, click **Edit**.

3. Click **Delete**.

4. Click **Really delete** to delete the course. The course is deleted and removed from the Courses list.


## Adding a cross-listed course

Cross-listed courses share instructors, course listing information, and reserve items. Once a course is created, cross-listed courses can be added to it. When you cross-list a course, the information you have in the original course’s Course listing information section also applies to the cross-listed course.

1. [Find the course](#searching-for-courses) you want to add a cross-listed course to and click on it in the **Courses** list.

2. In the **course details** window, click **Crosslist**.

3. In the **New course within listing** window, enter a **Course name** and optionally fill in the other boxes under **Basic course information**. The **Cross listing information** section is populated with information from the original course.

4. Click **Save & close**. The course is saved and appears in the Cross-listed courses section of the original course. It also appears in the main course list.


## Editing a cross-listed course

See [Editing a course.](#editing-a-course)


## Deleting a cross-listed course

You are able to delete a cross-listed course with items as long as one course remains.

1. [Find the cross-listed course](#searching-for-courses) you want to delete and click on it in the **Courses** list.

2. In the **course details** window, click **Edit**.

3. Click **Delete**.

4. Click **Really delete** to delete the course. The course is deleted and removed from the Courses list.


## Adding an instructor to a course

Instructors can only be added once a course is created. The instructor does not need a user record in FOLIO, but adding an instructor with a user record facilitates reports.

Add an instructor with a FOLIO user record:

1. [Find the course](#searching-for-courses) and click on it in the **Courses** list.

2. Under **Instructors**, click **Add instructor**.

3. In the **Add instructor** window, click **Look up user**.

4. In the **Select User** dialog, find the instructor you want to add, and click on them in the **User Search Results** list. The instructor’s name and barcode appears in the Name and Barcode boxes.

5. Click **Save & close**. The instructor appears in the Instructors section.


Add an instructor that does not have a FOLIO user record:

1. [Find the course](#searching-for-courses) and click on it in the **Courses** list.

2. Under **Instructors**, click **Add instructor**.

3. In the **Name** box, enter the instructor’s name.

4. Click **Save & close**. The instructor appears in the Instructors section.


## Editing an instructor

1. [Find the course](#searching-for-courses) and click on it in the **Courses** list.

2. Under **Instructors**, find the instructor you want to edit.

3. Click **Edit instructor**.

4. In the **Add instructor for [course]** window, edit the **Name** or **Barcode** of the instructor. 

5. Click **Save & close**.


## Deleting an instructor

1. [Find the course](#searching-for-courses) and click on it in the **Courses** list.

2. Under **Instructors**, find the instructor you want to delete.

3. Click **Remove**. The instructor is removed from the course.


## Adding a reserve item to a course

When you add an item to a course, the following information is copied from the original record: Title and Contributor from the Instance record; Barcode, Status, Permanent location, Copy, Volume, Enumeration, and URL/PDF link from the Item record; and Effective call number.

The Start date, End date, and Temporary location are automatically updated based on the Term and Location applied at the courses level. If you need to change these fields, or update reserve item level information, you will need to [edit the reserve item.](#editing-a-reserve-item)

1. [Find the course](#searching-for-courses) and click on it in the **Courses** list.

2. In the **Items** section, either scan the item barcode into the box, or enter the barcode and click **Add item**. The item is added to the course and appears in the Items section.


## Editing a reserve item

Note: If you add an item to a course and later make a change to the item via the item record (in the Inventory app) after that item is added to the course, then the change will not be reflected in the reserve record. To update the course reserve record, you need to delete the item and then re-add the item to the course.

Editing a reserve item allows you to change or add information to the following fields:

* **Temporary location.** If you change the reserve item’s temporary location, once you save the changes, the selected Temporary location is added to the Item record in the Inventory app.
* **Temporary loan type.** If you change the reserve item’s temporary loan type, once you save the changes, the selected Temporary loan type is added to the Item record in the Inventory app.
* **Processing status.** This field only applies to the Courses app and is available as a Reserves search filter.
* **Start Date and End Date.** When an item is placed on reserve, the start and end date are inherited from the selected Term.
* **Copyright information.** This section facilitates copyright tracking.

1. [Find the course](#searching-for-courses) with the item you want to edit and click on it in the **Courses** list.

2. In the **Items** section, find the reserve item and click **Edit reserve**.

3. In the **Item title** window, make your changes.

4. Click **Save & close**. The item is updated.


## Removing a reserve item from a course

Note: Removing an item from a course does not remove it from the Inventory app.

1. [Find the course](#searching-for-courses) with the item you want to remove and click on it in the **Courses** list.

2. In the **Items** section, find the reserve item and click **Remove**. The item is removed.