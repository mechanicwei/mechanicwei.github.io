---
title: 研读parallel_tests源码
date: 2016-03-21
tags:
---

> [parallel_tests-2.4.1](https://github.com/grosser/parallel_tests)

## rake parallel:spec

-   lib/tasks.rb, 建了rake任务，rake任务：执行`bin/parallel_rspec`
-   分组：  
    找到所有的测试文件。
    `Dir[File.join(folder, pattern)].uniq`

    通过`File.stat`，获取文件大小，然后进行排序

    ```ruby
        def largest_first(files)
            files.sort_by{ |_item, size| size }
        end
    ```

    每次把测试文件加入到最小的分组里面去

    ```ruby
        def group_features_by_size(items, groups_to_fill)
            items.each do |item, size|
                size ||= 1
                smallest = smallest_group(groups_to_fill)
                add_to_group(smallest, item, size)
            end
        end
    ```



- 分组并行执行测试  
    使用 [parallel](https://github.com/grosser/parallel) 这个gem来帮助完成  
    `Parallel.map(items, :in_threads => num_processes)`  
    但这里传的参数是`in_threads`，以多线程的方式运行，ruby里的多线程是不能达到并行的目的，让人有点费解。
    仔细思考下，这里是用`system()`来执行`rspec`命令

    > system() – Executes command… in a subshell.

    system 创建一个单独的进程来执行，从而达到并行测试的目的。

## Awesome

- 利用 block 计算方法执行时间
    ```ruby
        def delta
            before = now.to_f
            yield
            now.to_f - before
        end
    ```

- Enumerable#partition  
    `(1..6).partition { |v| v.even? } #=> [[2, 4, 6], [1, 3, 5]]`

- Enumerable#min_by

- OptionParser 命令行参数解析