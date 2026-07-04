---
layout: post
title: Text Classifier
description: A binary decision-tree classifier built from scratch in Java. It learns to label text (e.g. attributing Federalist Papers to their author, or sorting spam from real email) by recursively splitting on the word frequencies that best separate the training data.
skills: 
  - Java
  - Recursion
  - Binary trees
  - Machine learning basics

main-image: /tree.svg
---

## Overview

This project is a binary decision-tree classifier (from scratch).
Given labeled text examples, it builds a tree where each internal node asks a
yes/no question about a single feature (how frequently a given word appears), and each
leaf is a predicted label. To classify a new document, you walk from the root to a leaf,
turning left or right at each node depending on that document's word frequencies.

---

## The tree node

Each node is either a decision node (a feature + threshold, with left/right children) or
a leaf (a label, plus the training datapoint that created it). 
```java
private static class ClassifierNode {
    public final String label;
    public final String feature;
    public final double threshold;
    public final TextBlock data;

    public ClassifierNode left;
    public ClassifierNode right;

    // Decision node: split on 'feature' at 'threshold'.
    public ClassifierNode(String feature, double threshold) {
        this.feature = feature;
        this.threshold = threshold;
        this.label = null;
        this.data = null;
    }

    public ClassifierNode(String label) {
        this(label, null);
    }

    // Leaf node: a predicted 'label' and the datapoint that created it.
    public ClassifierNode(String label, TextBlock data) {
        this.label = label;
        this.data = data;
        this.feature = null;
        this.threshold = 0.0;
    }

    public boolean isLeaf() {
        return label != null;
    }
}
```

---

## Classifying: walk to a leaf

Classification is the simplest recursion: at a decision node, compare the input's value for
that node's feature against the threshold, and recurse into the matching child until you hit
a leaf.

```java
public String classify(TextBlock input) {
    if (input == null) {
        throw new IllegalArgumentException();
    }
    return classify(root, input);
}

private String classify(ClassifierNode node, TextBlock input) {
    if (node.isLeaf()) {
        return node.label;
    }
    if (input.get(node.feature) < node.threshold) {
        return classify(node.left, input);
    } else {
        return classify(node.right, input);
    }
}
```

---

## Training: growing the tree one example at a time

Training starts with a single leaf and feeds in examples one at a time. When an example
reaches a leaf whose label disagrees, that leaf is split: the feature with the biggest
difference between the two datapoints becomes a new decision node, with the two competing
labels as its children. The threshold is the midpoint of their values on that feature.

```java
private ClassifierNode train(ClassifierNode node, TextBlock current, String expectLabel) {
    if (node.isLeaf()) {
        if (node.label.equals(expectLabel)) {
            return node;
        }
        String feature = current.findBiggestDifference(node.data);
        double threshold = midpoint(current.get(feature), node.data.get(feature));
        ClassifierNode decision = new ClassifierNode(feature, threshold);
        ClassifierNode newLeaf = new ClassifierNode(expectLabel, current);
        if (current.get(feature) < threshold) {
            decision.left = newLeaf;
            decision.right = node;
        } else {
            decision.left = node;
            decision.right = newLeaf;
        }
        return decision;
    }
    if (current.get(node.feature) < node.threshold) {
        node.left = train(node.left, current, expectLabel);
    } else {
        node.right = train(node.right, current, expectLabel);
    }
    return node;
}
```

---

## Persistence: saving and reloading a trained model

A trained tree can be written to a file and rebuilt later, so a model doesn't have to be
retrained every run. `save` does a pre-order traversal; the matching constructor reloads it
with the same recursive shape.

```java
private void save(ClassifierNode node, PrintStream output) {
    if (node.isLeaf()) {
        output.println(node.label);
    } else {
        output.println("Feature: " + node.feature);
        output.println("Threshold: " + node.threshold);
        save(node.left, output);
        save(node.right, output);
    }
}

// Rebuilds a tree from a saved file, mirroring the pre-order format written by save().
private ClassifierNode load(Scanner input) {
    if (!input.hasNextLine()) {
        return null;
    }
    String line = input.nextLine();
    if (line.startsWith("Feature: ")) {
        String feature = line.substring("Feature: ".length());
        double threshold = Double.parseDouble(input.nextLine().substring("Threshold: ".length()));
        ClassifierNode node = new ClassifierNode(feature, threshold);
        node.left = load(input);
        node.right = load(input);
        return node;
    } else {
        return new ClassifierNode(line);
    }
}
```

The `save` and `load` formats are mirror images — `save` writes each subtree in pre-order,
and `load` reads it back in exactly the same order — which is what lets a model round-trip
to disk and back without losing its structure.
