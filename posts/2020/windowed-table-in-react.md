.. title: Windowed Table in React
.. slug: windowed-table-in-react
.. date: 2020-08-10 16:49:32 UTC+05:30
.. tags: ReactJS
.. author: Mayur Borse
.. link:
.. description:
.. type: text
.. summary: Rendering large number of rows in table is expensive task. What if we can implement a custom "Window" that renders fixed number of rows for any amount of data providing constant render. Let's check its implementation.

# Overview

It is very usual to come across requirement to render large sets of data on UI. In such case, as the data keeps growing so does our UI component (table/list) and this may cost us on performance side. Recenty, we at hyphenOs succeeded to achieve constant render for any number of data by implementing a custom Windowed Table component. We will check the use case and implementation details of this component in this post.

# Background

In our ongoing project `"webshark"`, we receive a network packets data on [webshark-frontend](https://github.com/hyphenOs/webshark-backend) sent by [webshark-backend](https://github.com/hyphenOs/webshark-backend) using `websocket` protocol. On reception, the packets are passed to `Table` component to render on UI. Though, it was doing the job well initially and for small amount of data, we realized the necessity to improve its overall performance for large data by making it "virtualized" or "windowed" Table. After initial try and test of available libraries we decided to create a custom "windowed" component. Let us introduce the contributing elements and the structure of the component.

# Table component

## Window Concept
