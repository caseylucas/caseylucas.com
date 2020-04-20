---
layout: post
title:  "Rendering gRPC Protobuf Files"
tags: grpc protobuf
---

I have used and been very happy with [gRPC](https://grpc.io) as an inter-service communication mechanism 
on multiple projects, however it took a few attempts to land on a protobuf rendering strategy that works well
for my typical use-case: numerous Go clients/services and TypeScript [gRPC-Web](https://github.com/grpc/grpc-web)
clients being implemented by multiple developers.  In this post I'll explain what has worked well.

**tl;dr**: Check out the example repo at 
[https://github.com/caseylucas/protobuf](https://github.com/caseylucas/protobuf).

## Requirements

There were a few requirements that evolved after having used gRPC in multiple projects:

1. **Simplify and standardize running protoc for numerous services.**
Sometimes it can feel like black magic getting protoc and it's command line options working - especially if 
multiple directories and languages are used. We want to get this right once and in one place and if we 
need to make changes options, do it in one place.

2. **Evolve protobuf definitions so that server implementations and client consumption can advance independently.**
The ability to evolve service definitions and implementations in a wire-compatible way is an inherent feature of gRPC
but most of the time the service and client do not evolve at the same time or necessarily in the same release.  So, 
it's important that the service implementation can depend on newer protobuf definitions while having the client depend
on slightly older definitions.

3. **Reuse protobuf definitions of common/shared messages across different services.**
Most services have a self-contained set of protobuf files but occasionally it makes sense to share the 
definition of a common message. If you don't reuse the definition at the protobuf level, you won't be able to 
easily share the implementation in the rendered code.

4. **Be notified of service definition changes without being as concerned with client/service implementation changes.**
I consider protobuf service definitions to be the contracts between clients and services and prefer to keep a
close eye on service changes that may affect numerous clients.

5. **Enforce a sane set of protobuf related conventions and styles.**
As projects evolve over time and multiple developers are creating and enhancing service definitions,
it would be great if tooling could help enforce some basic best practices or at least enforce
some consistency among service definitions.

6. **Package up rendered files so they can be easily used in both client and service implementations.**
The main languages we use are Golang and TypeScript and the "packaging" requirements for both these languages
are pretty minimal, however we found that using the gRPC-Web renderings with TypeScript works
best if we don't directly include rendered files in a TypeScript client project. Instead, you can include
them as a separate package.json dependency. Example issue:
[Failed to compile. 'proto' is not defined (also 'COMPILED')](https://github.com/grpc/grpc-web/issues/447)


## Prior Art

Of course I'm not the first person with similar requirements and was glad to find
[this post from the team at Namely](https://medium.com/namely-labs/how-we-build-grpc-services-at-namely-52a3ae9e7c35).
Indeed, parts of my solution are based on the script referenced in the post.
I recommend taking a look at their [docker-protoc repo](https://github.com/namely/docker-protoc).  At the time, 
when I first set things up a year or so ago, the docker-protoc functionality wasn't the best fit but it probably
warrants another look now.


## The Strategy

1. **Use a single repository to hold all protobuf files related to a set/domain of services.**
One repository helps with requirement 1 by centralizing the protoc settings and options. It also helps
with requirement 3 by keeping referenced protobuf files near each other and allowing them to
easily evolve together. Finally, we can set up a repository wide notification and be notified
of service definition related commits which helps with requirement 4.

2. **Use [Uber's prototool](https://github.com/uber/prototool) and run it via docker for consistent,
repeatable renderings.**
Prototool is a great tool for rendering protobuf files. You can have it enforce conventions (lint) and it
supports rendering multiple languages. Of course, using docker removes potential environment
inconsistency issues. Overall, using prototool helps with requirements 1, 3 and 5.

3. **Commit all rendered files for a particular service into their own language-specific repository.**
When creating a separate repository for each language and service pair, it becomes easy to evolve dependent
client and service implementations individually.  The clients and services simply use `yarn/npm` or `go get` to
include specific versions (usually latest/master) of rendered repositories. Using separate repositories for
rendered code helps with requirements 2 and 6.


## The Implementation

I'm a fan of `make`. It's a tried and true tool for pretty straight-forward cases. So we ended up with a makefile
that had targets like:

```
- generate: Runs prototool in order to validate / lint all *.proto files
- repos:    Create required protobuf-* github repos
- diff:     Show diff of *generated* code. does not commit changes - just shows diff
- commit:   Commits (and pushes) generated code (NOT *.proto files)
- clean:    Cleans up intermediate files
- help:     Print this help
```

You can see an example [Makefile](https://github.com/caseylucas/protobuf/tree/master/Makefile)
(which may need some customization) in this [example repo](https://github.com/caseylucas/protobuf).

## Use

Once set up, adding new services, editing existing ones and using the rendered code is pretty straightforward.

### New Service

1. Add new protobuf files for the new service - iteratively running `make generate` to work out the kinks in the
service definition.  You may need to modify `prototool.yaml` depending on the complexity of modifications.  You
can also run `make diff` if you really want to inspect the differences in rendered code.

2. Add a new definition for the rendered code repository to the `REPOS` variable in `rendered_repos.mk`.  See
[rendered_repos.mk](https://github.com/caseylucas/protobuf/blob/master/rendered_repos.mk#L3-L7) for examples.

3. Create new GitHub repositories (if they don't exist).
    ```bash
    make repos
    ```
This target wasn't strictly necessary but it conveniently creates new repositories and initializes Go module
support for rendered Go code.

4. Commit the rendered code to the language-specific repositories.  If `rendered_repos.mk` lists multiple languages
then they will all be committed.
    ```bash
    make commit 
    ```

5. Commit the modified protobuf files.  The commit make target doesn't also commit the current repository holding
protobuf files.  You'll need to do that with git.
    ```bash
    git commit
    ```

### Existing Service

Making modifications to an existing service definition:

1. Iteratively edit protobuf files and run `make generate`.

2. Once you're happy with the modifications, run `make commit` to publish the rendered code.

3. Don't forget to commit the protobuf definitions via `git commit`.

### Using the Rendered Code

Using the rendered code in other repositories holding the client and/or service implementations is typical 
for both Go:
```bash
go get -u github.com/caseylucas/protobuf-some_service-go
```

... and TypeScript:
```bash
yarn add https://github.com/caseylucas/protobuf-some_service-ts.git
```

Typically, the service implementation will evolve first. You can update the rendered service code and client code
independently.

## Wrap-up

I hope this helps someone that might be new to gRPC or otherwise struggling with getting protoc up and running.
Feedback is appreciated.

