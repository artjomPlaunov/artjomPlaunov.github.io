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

We use the Induction Principle, with the property

\\[   A(\phi) \equiv \varphi \in sub(\psi) \land \psi \in sub(\phi) \implies \varphi \in sub(\phi) \\]

(i) If $$\phi$$ is a propositional atom then $$\varphi$$ and $$\psi$$ of the induction hypothesis can only be equal to $$\phi$$, in which case $$\varphi = \phi \in \{\phi\} = sub(\phi)$$. (Since there is no other way for the implication to be true in this case).

(ii) In the second case, $$\phi$$ is of the form $$\phi_1 * \phi_2$$, where $$*$$ is a binary propositional connective. We assume the induction hypothesis for $$\phi_1$$ and $$\phi_2$$:

$$\begin{aligned} \varphi \in sub(\psi) \land \psi \in sub(\phi_1) \implies \varphi \in sub(\phi_1)\\ 
\varphi \in sub(\psi) \land \psi \in sub(\phi_2) \implies \varphi \in sub(\phi_2)\\
sub(\phi_1 * \phi_2) = sub(\phi_1) \cup sub(\phi_2) \cup \{  \phi_1 * \phi_2  \} 
\end{aligned}$$