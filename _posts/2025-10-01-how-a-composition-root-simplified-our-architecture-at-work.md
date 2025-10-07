How a Composition Root Simplified Our Architecture at Work

When developing software features, it’s easy to fall into the trap of short-term decisions that later become long-term complexity. At first, everything seems manageable—requirements are clear enough, dependencies are minimal, and the code flows logically. But as features evolve and start interacting, things can get messy fast.
This is a story of how I used the composition root pattern to untangle a growing web of features in a module I was working on—and how it helped restore clarity and reusability to our codebase.

The Problem: Three Entry Points, Three Versions of the Same Thing
In our module, we had three separate entry points, each responsible for initializing a different feature. Initially, this seemed fine. Each feature was isolated, had its own flow, and created its own dependencies. But over time, we noticed a pattern:
* These features were very similar, almost duplicates in logic and structure.
* Each feature created its own dependencies, which worked when they were the only ones using them.
* But when we tried to reuse a dependency across features, things broke down. Dependencies were tightly coupled to their parent flows, and reusability became a nightmare.
<img width="1013" height="1002" alt="Screenshot 2025-10-07 at 15 27 32" src="https://github.com/user-attachments/assets/913614fd-5dac-4abe-9af3-e7ef23f55198" />

The Tangle: Hidden Coupling and Duplication
The root of the problem was that each feature was acting like its own little application. They instantiated their own dependencies, unaware of the bigger picture. This led to:
* Duplication of logic and configuration.
* Hidden coupling, where changes in one feature’s dependency setup affected others in unpredictable ways.
* Reduced testability, since mocking or replacing dependencies required deep changes in each feature’s flow.

The Shift: One Entry Point to Rule Them All
My first instinct was to consolidate the entry points. Instead of having three separate flows, we’d have one composition root—a single place where all dependencies are created and wired together.
This change had a few immediate benefits:
* Centralized configuration: All dependencies were created in one place, making it easier to manage and understand.
* Improved reusability: Dependencies could be shared across features without duplication.
* Cleaner architecture: Features became consumers of dependencies, not creators.
<img width="1013" height="1031" alt="Screenshot 2025-10-07 at 15 27 37" src="https://github.com/user-attachments/assets/d7d9671b-2c34-4ee2-80be-041e2be7c034" />

The Composition Root: A Better Way to Wire Things Together
The composition root is a concept from dependency injection. It’s the place in your application where you compose the object graph—where you create and connect all your services, repositories, and other dependencies.
By moving dependency creation to the composition root, we achieved:
* Decoupling: Features no longer cared how their dependencies were created.
* Flexibility: We could swap implementations, mock services, or add decorators without touching feature code.
* Scalability: Adding new features became easier, since they could plug into the existing dependency graph.
<img width="970" height="997" alt="Screenshot 2025-10-07 at 15 27 47" src="https://github.com/user-attachments/assets/f649055e-a795-43ad-981e-e13fd15ba5f1" />

Lessons Learned
Here are a few takeaways from this experience:
1. Avoid premature encapsulation: Isolating features too early can lead to duplication and coupling.
2. Centralize dependency creation: A composition root helps maintain clarity and control.
3. Think in terms of flows and reuse: Features should be composable, not siloed.
<img width="1000" height="922" alt="Screenshot 2025-10-07 at 15 35 14" src="https://github.com/user-attachments/assets/e33fd174-c068-4123-a2a9-ca34720dcc53" />

Final Thoughts
Untangling features isn’t just about refactoring—it’s about rethinking how your application is structured. The composition root gave us a way to step back, see the bigger picture, and build a more maintainable, scalable system.
If you’re struggling with feature duplication or tangled dependencies, consider introducing a composition root. It might just be the architectural reset your project needs.
