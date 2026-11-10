# Testing and Guardrails: Validating AI-Generated Code

:::{admonition} Overview
:class: note
**Questions**

* How can we trust LLM-generated code for scientific and physics applications?
* How do we prevent AI models from silently hallucinating math or physics logic?

**Objectives**

* Understand why Test-Driven Development (TDD) is highly recommended when using AI agents.
* Write a physics-informed test using pytest to validate LLM output against known textbook results.
:::

## The Trust Problem in Scientific AI

Large Language Models (LLMs) are highly capable of writing syntactically correct Python or C++ code. However, they lack intrinsic physics intuition. A model might generate code that runs perfectly without errors but calculates a cross-section, invariant mass, or momentum vector incorrectly.

To prevent this, we should use **Scientific-informed guardrails**. Before asking the AI to write a function, the user must define the constraints using a testing framework like pytest.

## Test-Driven Development (TDD) with LLMs

As an example, below is a common High-Energy Physics (HEP) scenario: calculating the invariant mass of a particle from two daughter four-vectors.

Instead of asking the LLM to write the code first, we write the test first based on a known physics truth.

```python
# test_kinematics.py
import pytest
import numpy as np
import vector
# Assume the LLM will write a function called calculate_invariant_mass in kinematics.py
from kinematics import calculate_invariant_mass

def test_invariant_mass():
    # Known textbook case: two massless photons moving in opposite directions
    # with E = 10 GeV.
    # The invariant mass should be exactly 20 GeV.

    # Particle 1
    p1 = vector.obj(px=0.0, py=0.0, pz=10.0, E=10.0)

    # Particle 2 (Opposite direction)
    p2 = vector.obj(px=0.0, py=0.0, pz=-10.0, E=10.0)

    # Passing 2 vector objects instead of 8 separate float arguments
    mass = calculate_invariant_mass(p1, p2)

    assert np.isclose(mass, 20.0, rtol=1e-3), f"Expected 20.0, but got {mass}"
```

:::{admonition} Exercise: Generating and Validating the Code
:class: tip

1.  Save the code above in a file called test_kinematics.py.
2.  Open your preferred AI assistant (ChatGPT, Claude, or an agentic IDE).
3.  Prompt the AI: "Write a Python function calculate_invariant_mass(p1, p2) that takes two Scikit-HEP vector objects and calculates their invariant mass. Return only the code."
4.  Save the AI output in a file called kinematics.py.
5.  Run the test in your terminal by executing pytest test_kinematics.py.

Did the AI get the physics right on the first try? :::

::::{admonition} Solution
:class: dropdown
Often, standard LLMs might use naive Euclidean geometry instead of the Minkowski metric formulas (i.e., they might add the squares instead of subtracting the momentum squared from the energy squared), causing the test to fail.

If the test fails, do not fix the code yourself. Instead, use the test failure as a prompt to create an iterative feedback loop:

"Your code failed this pytest with the error: AssertionError: Expected 20.0, but got 0.0. You likely used standard spatial distance instead of the relativistic invariant mass formula. Please correct the calculation."
::::

:::{admonition} Key Points
:class: important

  - The human is always responsible for the correctness of scientific code.
  - Never deploy AI-generated physics functions without automated unit tests checking edge cases and known limits.
  - Use pytest output as direct feedback to prompt the LLM to fix its own hallucinations.
:::
