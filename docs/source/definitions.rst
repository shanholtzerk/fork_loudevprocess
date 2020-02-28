.. _glossary:

Definitions
===============



.. glossary::
    :sorted:

    fork
        :term:`git` term. A fork is a personal copy of another user's :term:`repository` that lives on your account.
        Forks allow you to freely make changes to a project without affecting the original. Forks remain attached to
        the original, allowing you to submit a pull request to the original's author to update with your changes. You
        can also keep your fork up to date by pulling in updates from the original.

    git
        configuration management system. See also
        https://help.github.com/en/github/getting-started-with-github/github-glossary and
        https://mirrors.edge.kernel.org/pub/software/scm/git/docs/gitglossary.html

    origin
        :term:`git` term. The default upstream :term:`repository`. Most projects have at least one upstream project which
        they track. By default origin is used for that purpose. New upstream updates will be fetched into
        remote-tracking branches named origin/name-of-upstream-branch, which you can see using git branch -r.

    remote
        :term:`git` term. This is the version of something that is hosted on a server, most likely GitHub. It can be
        connected to local clones so that changes can be synced.

    repository
        :term:`git` term. A repository is the most basic element of GitHub. They're easiest to imagine as a project's
        folder. A repository contains all of the project files (including documentation), and stores each file's
        revision history. Repositories can have multiple collaborators and can be either public or private.

    upstream
        :term:`git` term. When talking about a branch or a :term:`fork`, the primary branch on the original
        :term:`repository` is often referred to as the "upstream", since that is the main place that other changes
        will come in from. The branch/fork you are working on is then called the "downstream".