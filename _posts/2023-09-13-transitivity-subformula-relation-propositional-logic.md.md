---
layout: post
title:  Transitivity of the Subformula Relation for Propositional Logic
date:   2023-09-13
description: Exercise 2.1.3 from Dalen's "Logic and Structure", 5th edition.
tags: PropositionalLogic, Transitivity, Subformula
categories: logic
---
Problem: Show that the relation "is a subformula of" is transitive.

**Definition 2.1.7** The set of subformulas \\(Sub(\phi)\\) is given by

$$\begin{aligned} Sub(\phi) &= \{ \phi \} \text{ for atomic } \phi \\ 
Sub(\phi_1 * \phi_2) &= Sub(\phi_1) \cup Sub(\phi_2) \cup \{\phi_1 * \phi_2\} \\
Sub(\neg \phi) &= Sub(\phi) \cup \{\neg \phi\} \\
\end{aligned}$$
