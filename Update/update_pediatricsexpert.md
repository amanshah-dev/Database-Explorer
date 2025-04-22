create procedure update_pediatricsexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Pediatrics', achievementCategoryId, userIds);
END;
$$;

alter procedure update_pediatricsexpert(bigint, bigint[]) owner to learner;
