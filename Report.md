# Computational Logic for Artificial Intelligence Assignment Report

The aim of this assignment was to add to the functionality of the Prolexa code by addressing some of the logic that it couldn’t currently handle. This implementation added to Prolexa by allowing for negation, existential quantification, and default reasoning. This report will briefly outline how these were implemented, how they can be used in Prolog, and their short-comings. 

## Negation

Negation was the first functionality implemented. The original Prolexa could not handle negation so did not understand logic such as: ‘If X then Y, not Y, therefore not X.’ A real world example of this would be: ‘All puppies are cute, Donald is not cute, therefore Donald is not a puppy.’

This example can be used to test the new functionality in the code. The issue with Prolog is that it uses negation as failure which is not the same as logical negation. Negation as failure can determine that something isn't true because it hasn't been explicitly stated. For example, in the case of ‘cute if puppy’, Prolog will look for a puppy and this rule will not be relevant for a fact that says X is not a puppy. It won't infer from this that X isn't cute. It would be a lazy fix to just flip the rule and say something like ‘not teacher if not happy’ but just putting this in the rules isn't generalisable and it wouldn't make sense to say 'every not happy is not a teacher' as a proof but it does make sense to say 'every teacher is happy'. 

Logical negation was tested using the following stored rule added to the Prolexa code (note that all stored rules are commented out for the final submission to allow for full testing of functionality):
```
stored_rule(1,[(happy(X):-teacher(X))]).
stored_rule(1,[(not happy(donald):-true)]).
stored_rule(1,[(cute(X):-puppy(X))]).
stored_rule(1,[(not cute(donald):-true)]).
```
The rule 'stored_rule(1,[(cute(X):-puppy(X))]).' logically means that every puppy is cute. What it doesn't mean is that everything that is cute is a puppy (e.g., a kitten is also cute) which means that the implication doesn't go both ways hence why the prove_rb predicate doesn't prove both ways.

Two unary predicates were added to the grammar for this rule which were ‘puppy’ and ‘cute’. Two new sentence structures were also added to the grammar as seen in the code below. The first case allows for sentences such as declaring ‘Donald is not a teacher’. The second case allows for declarations such as ‘All birds are not happy’. This latter example isn’t extremely useful as the meta-interpreter still can’t handle universal negations. So, ‘every teacher is happy’ works but ‘every teacher is not happy’ would be added to the grammar but wouldn’t be explainable. 
```
sentence1([(not L:-true)]) --> proper_noun(N,X),verb_phrase(N,not X=>L). 
sentence1([(not H:-B)]) --> determiner(N,M1,M2,[(not H:-B)]),noun(N,M1),verb_phrase(N,not M2).
```
Other small additions to the grammar were added such as negations in verb phrases, e.g., ‘is not’, ‘are not’, etc.

To enable negation proofs, prove_question and explain_question had an alternative added so that if a query fails it will attempt the negation of that query. So, in the case where every teacher is happy and Donald is not happy, a question of ‘is Donald a teacher’ would first attempt teacher(donald) which would fail and would then attempt not teacher(donald) which would succeed. The following simple addition to the meta-interpreter was added to handle negation. It simply finds a rule such as ‘happy if teacher’ and proves not teacher by recursively calling the predicate with not happy.
```
prove_rb(not B,Rulebase,P0,P):- % Added for negation
    find_clause((A:-B),Rule,Rulebase),
    prove_rb(not A,Rulebase,[p(not B,Rule)|P0],P).
```
### Testing:

The following is a test of negation in Prolog:

> “Every puppy is cute”.

> I will remember that every puppy is cute

> “Donald is not cute”.

> I will remember that donald is not cute

> “Is Donald a puppy”.

> donald is not a puppy

> “Explain why Donald is not a puppy”.

> donald is not cute; every puppy is cute; therefore donald is not a puppy

This proves that logical negation has been implemented. The short-coming of this implementation is negations in defined rules. For example:

> “Every teacher is not happy”.

> I will remember that every teacher is not happy

> “Donald is happy”.

> I will remember that donald is happy

> “Is Donald a teacher”.

> Sorry, I don't think this is the case

This wouldn’t necessarily be a hard thing to implement but was deemed not important for the scope of this assignment to allow for other functionalities to be added. This extension could be seen as future work.

## Existential Quantification

Existential quantification was added to be able to handle cases such as ‘some humans are geniuses; geniuses win; therefore, some humans win.’

To be able to add this functionality, Skolemisation needs to be used which essentially eliminates existential quantification in full clausal logic. This means that a new type of phrase would have to be defined in the grammar to allow for new rules to be added, questions to be asked, and for explanations. The following rules were added for testing this functionality. The first rule is an existential quantification representing the logic of ‘some humans are geniuses’.
```
stored_rule(1,[(human(sk):-true), (genius(sk):-true)]).
stored_rule(1,[(win(X):-genius(X))]).
stored_rule(1,[(genius(donald):-true)]).
```
In the grammar ‘human’, ‘genius’ and ‘win’ are defined. The ‘geniuses’ plural form of ‘genius’ was also added as a case in the noun_s2p predicate as this does not follow the general rule of adding an ‘s’ for a plural (‘wins’ on the other hand does follow this rule so doesn’t need to be defined manually). 

The following sentence case was added for existential quantification which essentially transforms the clause into the correct format for an existential quantification clause. 
```
sentence1([([H1,H2]:-true)]) --> determiner(N,M1,M2,[(H1:-true),(H2:-true)]),noun(N,M1),verb_phrase(N,M2). 
```
The following two determiners were uncommented from the original code and are used for existential quantification such as ‘some humans are geniuses’.
```
determiner(p, sk=>H1, sk=>H2, [(H1:-true),(H2 :- true)]) -->[some].
determiner(p, sk=>H1, sk=>H2, [(H2:-true),(H1:-true)]) -->[some].
```
The following two questions were added to the grammar to allow for questions such as ‘do some humans win’ and ‘are some humans happy’. 
```
question1((Q1,Q2)) --> [do],[some],noun(p,sk=>Q1),verb_phrase(p,sk=>Q2).
question1((Q1,Q2)) --> [are],[some],noun(p,sk=>Q1),property(p,sk=>Q2).
```
The final addition to the grammar for existential quantification was the following command to allow for explanations of existential quantification such as ‘explain why some humans win’. 
```
command(g(explain_question([Q1,Q2],_,Answer),Answer)) --> [explain],[why],sentence1([(Q1:-true),(Q2:-true)]).
```
The addition to the prolexa engine for existential quantification was very simple and followed parts of the code in Section 7.3 (pages 145-146) of Peter Flach’s simply logical book. The following was added to the meta-interpreter to allow for the proof of the two different parts of the existential quantification. This is the same as the case for explain_rb which was implemented for default reasoning but it can be seen that it follows the same  code structure for proof and explanation. 
```
prove_rb((A,B),Rulebase,P0,P):-!,
    prove_rb(A,Rulebase,P0,P),
    prove_rb(B,Rulebase,P0,P).
```

```
explain_rb((A,B),Rulebase,P0,P):-!,
  explain_rb(A,Rulebase,P0,P1),
  explain_rb(B,Rulebase,P1,P).
```
The following find_clause predicate was added from the code in Section 7.3 which copies an element of the clause instead of directly copying a term from the rule. 
```
find_clause(Clause,Rule,[Rule|_Rules]):-
    copy_element(Clause,Rule).
```
### Testing:

Up-front, although the logic of existential quantification works, there is a minor issue in the explanation of this logic as will be seen in the following example:

> “Some humans are geniuses”.

> I will remember that some humans are geniuses

> “Every genius wins”.

> I will remember that every genius wins

> “Do some humans win”.

> some humans win

> “Explain why some humans win”.

> some humans are geniuses; every genius wins; some humans are geniuses; therefore some humans win

It can be seen that the proof goes in a small loop and ends up repeating the existential quantification rule of ‘some humans are geniuses’. This is not a deal-breaker as the logic is still sound and produces the desired outcome but does make the explanation messy. Again, in the interest of spending time implementing other functionalities rather than fixing this, the problem was left as future work.

## Default reasoning 

The last functionality implemented was default reasoning and the code is almost identical to that in Section 8.1 (pages 153-154) of Peter Flach’s simply logical book. The following rules were added for testing default reasoning. Not much had to be added to the grammar for this functionality. All necessary words were included in the grammar such as ‘bird’, ‘penguin’, etc.
```
stored_rule(1,[(bird(X):-penguin(X))]). 
stored_rule(1,[(bird(X):-robin(X))]).
stored_rule(1,[(penguin(tweety):-true)]).
stored_rule(1,[(robin(rob):-true)]).
stored_rule(1,[(default(fly(X):-bird(X)))]).
stored_rule(1,[(not fly(X):-penguin(X))]). % exception to the default rule
```
The important addition to the grammar was the following determiner which sets up default rules for cases where the word ‘most’ is used. The word ‘most’ was used purely to distinguish default reasoning from existential quantification which uses the ‘some’ determiner. This determiner allows for statements such as ‘most birds fly’. 
```
determiner(p,X=>B,X=>H,[(default(H:-B))]) --> [most]. 
```
To allow for the code from Section 8.1 to be used in the engine code, the prove_question and explain_question predicates use explain_rb instead of prove_rb. The code for explain_rb is fairly simple as it still allows for explanation by rules only as would have happened previously but it also allows for default clauses to be found and explained. 
```
explain_rb(true,_Rulebase,P, P):-!.
explain_rb((A,B),Rulebase,P0,P):-!,
  explain_rb(A,Rulebase,P0,P1),
  explain_rb(B,Rulebase,P1,P).
explain_rb([A,B],Rulebase,P0,P):-!,
  explain_rb(A,Rulebase,P0,P1),
  explain_rb(B,Rulebase,P1,P).
explain_rb(A,Rulebase,P0,P):-
  prove_rb(A,Rulebase,P0,P). % explain by rules only
explain_rb(A,Rulebase,P0,P):-
    find_clause(default(A:-B),Rule,Rulebase),
  explain_rb(B,Rulebase,[p(A,Rule)|P0],P),
  not contradiction(A,Rulebase,P). 
```
This top-level proof was added and provides the same functionality as the prove_rb one in the original code in that it simply ignores proofs. 
```
explain_rb(Q,RB):-
    explain_rb(Q,RB,[],_P).
```
The following meta-interpreter for rules and contradiction checking were also added again simply to follow the code in Section 8.1.  
```
% Meta-interpreter for rules
prove_e(true,_Rulebase,P,P):-!.
prove_e((A,B),Rulebase,P0,P):-!,
    find_clause((A:-C),Rule,Rulebase),
    conj_append(C,B,D),
  prove_e(D,Rulebase,[p((A,B),Rule)|P0],P).
prove_e(A,Rulebase,P0,P):-
  find_clause((A:-B),Rule,Rulebase),
    prove_e(B,Rulebase,[p(A,Rule)|P0],P).

% Check contradiction against rules
contradiction(not A,Rulebase,P):-!,
    prove_e(A,Rulebase,P,_P1).
contradiction(A,Rulebase,P):-
    prove_e(not A,Rulebase,P,_P1).
```
Another important addition was the following code as an alternate case for the known_rule predicate. Without this, multiple instances of the same default rule could be added such as ‘most birds fly’ and ‘most birds fly’. Previously, this would not have been flagged as a known rule, but with this code addition the shell would return this as an already known rule. 
```
try((numbervars(Rule,0,_),
             Rule=default(H:-B), 
             add_body_to_rulebase(B,Rulebase,RB2),
             prove_rb(H,RB2)
        ))
```

### Testing:

Again, up-front there is a short-coming with the way the proofs and explanations are returned even though the logic is sound. Similarly to existential quantification, this is best described using an example:

> “Penguins are birds”.

> I will remember that penguins are birds

> “Robins are birds”.

> I will remember that robins are birds

> “Tweety is a penguin”.

> I will remember that tweety is a penguin

> “Rob is a robin”.

> I will remember that rob is a robin

> “Most birds fly”.

> I will remember that most birds fly

> “Penguins do not fly”.

> I will remember that penguins do not fly

> “Does Tweety fly”

> tweety does not fly

> “Explain why Tweety does not fly”.

> tweety is a penguin; every penguin does not fly; therefore tweety does not fly

> “Does Rob fly”.

> rob flies

> “Explain why Rob flies”.

> rob is a robin; every robin is a bird; most birds fly; therefore rob flies

This example shows that the logic is sound but the response isn’t ideal. The response would ideally be ‘tweety is a penguin; every penguin is a bird; most birds fly except penguins; therefore tweety does not fly’. The ideal response for Rob the robin would be ‘rob is a robin; every robin is a bird; most birds fly except penguins; therefore rob flies”. 

This short-coming is an issue with the grammar and the method of finding proofs. In the interest of time this issue was not further explored but would make interesting future work. 

In conclusion, three functionalities have been added which have logically allowed for negation, existential quantification, and default reasoning. All short-comings of these implementations would make for interesting future work to extend the useability of these functions and the clarity of their explanations.
