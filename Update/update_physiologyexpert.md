create procedure update_physiologyexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Physiology', achievementCategoryId, userIds); -- Call the main procedure with 'Physiology'
END;
$$;

alter procedure update_physiologyexpert(bigint, bigint[]) owner to learner;
